---
layout: post
title: Effective C++ 条款52：写了placement new也要写placement delete
categories: [阅读笔记, Effective C++]
tag: [Effective C++]
date: '2025-07-31 20:26:46 +0800'
---

## **Effective C++ 条款52 ：写了*placement* new也要写*placement* delete**

---

<br/>

### ⚙️ **一、Placement New/Delete 的本质与作用**

#### 1. **标准 `new operator` 的调用流程**  
当执行 `new` 表达式时（如 `T* p = new T;`），实际分为两步：
- **步骤1**：调用 `operator new` 分配内存（可能自定义）。
- **步骤2**：在分配的内存上调用构造函数。  
若步骤2（构造函数）抛出异常，**运行期系统需自动释放步骤1分配的内存**（调用`operate delete`），否则内存泄漏。

#### 2. **Placement New 的定义**  

> 类内定义的 `placement new` ​不会被普通 `new operator` 自动调用，仅在显式使用 `new (addr) Type` 语法时触发
{: .prompt-tip}

任何接受**额外参数**的 `operator new` 都称为 **placement new**。最常见的是定位构造版本：
```cpp
// 标准库提供的 placement new（用于定位构造）
void* operator new(std::size_t, void* p) noexcept { 
    return p; // 直接返回传入指针
}
```
自定义版本示例（带日志参数）：
```cpp
class Widget {
public:
    static void* operator new(std::size_t size, std::ostream& log) {
        log << "Allocating " << size << " bytes\n";
        return ::operator new(size); // 转交全局operator new
    }
};
```

#### 3. **Placement Delete 的匹配规则**  
运行期系统在构造函数异常时，会调用与 `operator new` **参数完全匹配**的 `operator delete`：
- 若未定义匹配的 `operator delete`，则**无法释放内存**，导致泄漏。

---

### ⚠️ **二、未配对的风险：隐蔽的内存泄漏**

#### **问题场景**  
```cpp
class Widget {
public:
    static void* operator new(std::size_t size, std::ostream& log);
    static void operator delete(void* p) noexcept; // 缺少 placement delete 版本
};

Widget* p = new (std::cerr) Widget; // 若构造函数抛出异常
```
- 运行期系统需调用 `operator delete(void*, std::ostream&)` 释放内存，但该类未提供此版本。
- 编译器**不会报错**，而是**静默跳过释放操作**，导致内存泄漏。

---

### 🛠️ **三、正确实现：成对声明与避免名称遮掩**

> **注意**：Placement delete **仅在构造函数异常时由系统调用**，手动 `delete(p)` 始终调用标准 `operator delete`。
{: .prompt-warning}

#### 1. **显式声明配对版本**  

```cpp
class Widget {
public:
    // 标准 new/delete
    static void* operator new(std::size_t size);
    static void operator delete(void* p) noexcept;

    // Placement new/delete
    static void* operator new(std::size_t size, std::ostream& log);
    static void operator delete(void* p, std::ostream& log) noexcept;
};
```

#### 2. **处理名称遮掩问题**  
类内声明任何 `operator new` 会**遮掩全局版本**（包括标准 `new`）：
```cpp
class Base {
public:
    static void* operator new(std::size_t size, std::ostream& log);
};
Base* p1 = new Base; // 错误！标准 new 被遮掩
Base* p2 = new (std::cerr) Base; // 正确
```
**解决方案**：使用继承基类引入标准版本，并通过 `using` 暴露：
```cpp
class StandardNewDeleteForms {
public:
    // 标准版本
    static void* operator new(std::size_t size) { return ::operator new(size); }
    static void operator delete(void* p) noexcept { ::operator delete(p); }
    // 标准 placement new（定位构造）
    static void* operator new(std::size_t size, void* ptr) { return ::operator new(size, ptr); }
    static void operator delete(void* p, void* ptr) noexcept { ::operator delete(p, ptr); }
};

class Widget : public StandardNewDeleteForms {
public:
    using StandardNewDeleteForms::operator new; // 暴露标准版本
    using StandardNewDeleteForms::operator delete;
    
    // 自定义 placement new
    static void* operator new(std::size_t size, std::ostream& log);
    static void operator delete(void* p, std::ostream& log) noexcept;
};
```

---

### 💡 **四、实践技巧与常见误区**

#### 1. **Placement Delete 的实现内容**  
- 通常与标准 `delete` 逻辑相同（如 `::operator delete(p)`），但需匹配额外参数：
  ```cpp
  void Widget::operator delete(void* p, std::ostream& log) noexcept {
      log << "Freeing memory due to ctor exception\n";
      ::operator delete(p); // 实际释放
  }
  ```

#### 2. **区分使用场景**  

| **操作类型**           | **调用时机**         | **是否需要自定义**            |
| ---------------------- | -------------------- | ----------------------------- |
| **标准 `new/delete`**  | 正常构造/析构        | 是（类专属优化时）            |
| **Placement `new`**    | 定位构造或带参分配   | 是（需额外逻辑时）            |
| **Placement `delete`** | **仅构造函数异常时** | **必须与 placement new 配对** |
| **手动 `delete`**      | 显式调用 `delete`    | 调用标准版本                  |

#### 3. **避免的陷阱**  

- **误区**：在 placement delete 中释放非内存资源（如文件句柄）。  
  **正解**：资源释放应通过析构函数完成，placement delete **仅处理内存**。
- **线程安全**：若 placement new 涉及全局状态（如日志流），需考虑锁机制。

---

### 💎 **总结：关键原则**

1. **成对实现**：自定义 placement new **必须**提供相同参数的 placement delete。
2. **暴露标准版本**：通过继承基类 + `using` 声明避免遮掩全局 `operator new/delete`。
3. **构造函数异常安全**：仅依赖系统自动调用 placement delete，**无需手动干预**。
4. **与条款49/51的关系**：  
   - 条款49（`new-handler`）处理分配失败，条款52处理分配成功但构造失败。  
   - 条款51（自定义`new/delete`）需包含对 placement 版本的支持。

> **Scott Meyers 强调**：Placement new/delete 的配对是C++内存管理中“**沉默的契约**”，违反它会导致难以调试的间歇性泄漏。实践中优先使用标准库设施（如`std::make_shared`）可减少此类风险。
