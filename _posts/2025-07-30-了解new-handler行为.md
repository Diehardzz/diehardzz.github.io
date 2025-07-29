---
layout: post
title: Effective C++ 条款49：了解new-handler行为
categories: [阅读笔记, Effective C++]
tag: [Effective C++]
date: '2025-07-30 03:45:00 +0800'
---

## **Effective C++ 条款49 ：了解new-handler行为**

---

<br/>

### ⚙️ 一、`new-handler`机制的原理与作用

1. **`operator new`失败时的默认行为**  
   当`operator new`无法满足内存请求时：
   - **旧式编译器**：返回`nullptr`（已过时）。
   - **现代C++**：抛出`std::bad_alloc`异常。  
   在抛出异常前，会先调用**客户指定的错误处理函数**——即`new-handler`。

2. **`new-handler`的定义与设置**  
   - **类型**：`std::new_handler`是函数指针类型：`typedef void (*new_handler)();`。
   - **设置方法**：通过`std::set_new_handler`注册自定义函数：
     ```cpp
     void CustomHandler() { 
         std::cerr << "Memory allocation failed!";
         std::abort(); 
     }
     int main() {
         std::set_new_handler(CustomHandler); // 注册处理函数
         int* p = new int[1e15]; // 触发失败，调用CustomHandler
     }
     ```
     若未设置`new-handler`，则直接抛出`std::bad_alloc`。

---

### 🛠️ 二、设计有效的`new-handler`策略

一个合理的`new-handler`应至少实现以下**策略之一**：  
1. **释放可用内存**  
   - 预先分配备用内存池，在触发`new-handler`时释放，使后续分配可能成功。
2. **切换处理函数**  
   - 动态替换`new-handler`（如通过全局变量控制），尝试不同恢复策略。
3. **卸载自身并退出**  
   - 调用`std::set_new_handler(nullptr)`卸载当前处理函数，让后续失败直接抛异常。
4. **终止程序**  
   - 调用`std::abort()`或`std::exit()`，避免程序进入不可预测状态。

---

### 🧩 三、实现类专属`new-handler`（核心技巧）

C++不支持类级别的`new-handler`，但可通过**静态成员与RAII**模拟该行为：  
#### **方法1：手动管理每个类的处理函数**  
```cpp
class Widget {
public:
    static std::new_handler set_new_handler(std::new_handler p);
    static void* operator new(std::size_t size);
private:
    static std::new_handler currentHandler; // 类专属处理函数
};

std::new_handler Widget::currentHandler = nullptr;

void* Widget::operator new(std::size_t size) {
    NewHandlerHolder h(std::set_new_handler(currentHandler)); // RAII包装器
    return ::operator new(size); // 调用全局operator new
}
```
**关键点**：  
- **RAII类`NewHandlerHolder`**：在构造时保存旧处理函数，析构时自动恢复，避免异常导致状态残留。
- **静态成员`currentHandler`**：存储类专属的处理函数指针。

#### **方法2：模板化实现（推荐）**  
```cpp
template<typename T>
class NewHandlerSupport {
public:
    static std::new_handler set_new_handler(std::new_handler p);
    static void* operator new(std::size_t size);
private:
    static std::new_handler currentHandler;
};

template<typename T>
std::new_handler NewHandlerSupport<T>::currentHandler = nullptr;

class Widget : public NewHandlerSupport<Widget> {}; // 继承模板
```
**优势**：  
- **类型参数`T`**：为每个派生类生成独立的`currentHandler`静态实例，避免手动重复实现。
- **复用性**：任何类只需继承`NewHandlerSupport`即可获得专属`new-handler`支持。

---

### ⚠️ 四、注意事项与陷阱

1. **`nothrow new`的局限性**  
   `new (std::nothrow)`仅抑制内存分配阶段的异常，**后续构造函数仍可能抛出异常**，因此无法完全避免异常处理。
2. **处理函数中的内存分配**  
   在`new-handler`内部**避免触发二次内存分配**，否则可能陷入无限递归或直接终止。
3. **多线程安全**  
   静态成员`currentHandler`需考虑线程同步（如`std::atomic`或互斥锁），但条款未深入讨论，实践中需额外处理。

---

### 💎 五、总结与最佳实践

| **关键点**       | **实践建议**                                              |
| ---------------- | --------------------------------------------------------- |
| **理解默认行为** | 内存失败时先调用`new-handler`，未设置则抛`std::bad_alloc` |
| **设计处理策略** | 优先实现“释放备用内存”或“切换处理函数”，避免无效操作      |
| **类专属处理**   | 使用模板类`NewHandlerSupport`实现零重复代码               |
| **资源管理**     | 用RAII（如`NewHandlerHolder`）确保处理函数状态安全恢复    |

> Scott Meyers强调：`new-handler`机制是C++对内存控制权的移交，**理解其递归调用本质**（处理函数可能被多次调用）是避免死循环的关键。 实践中应结合业务场景选择策略，如实时系统可能直接终止，而服务端程序可能尝试释放缓存。
