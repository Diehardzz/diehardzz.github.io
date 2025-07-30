---
layout: post
title: Effective C++ 条款51：编写new和delete时需固守常规
categories: [阅读笔记, Effective C++]
tag: [Effective C++]
date: '2025-07-30 20:01:01 +0800'
---

## **Effective C++ 条款51 ：编写new和delete时需固守常规**

---

<br/>

> Effective C++条款51的核心在于**安全实现自定义内存管理函数**（`operator new`/`operator delete`）时需遵循的关键规则。这些规则确保自定义内存管理器与C++标准行为兼容，避免资源泄漏、未定义行为及继承体系中的隐患。
{: .prompt-info}

### ⚙️ **一、`operator new`的实现规范**

#### 1. **处理0字节请求**

   - **标准要求**：C++规定即使请求0字节，`operator new`也必须返回合法指针（不能返回`nullptr`）。
   - **实现策略**：将0字节请求视为1字节请求：
     ```cpp
     void* operator new(std::size_t size) throw(std::bad_alloc) {
         if (size == 0) size = 1;  // 0字节视为1字节
         while (true) {
             void* ptr = malloc(size);
             if (ptr) return ptr;
             // 分配失败时调用new-handler（见下文）
         }
     }
     ```

#### 2. **处理继承体系中的大小不匹配**

   - **问题**：基类的`operator new`可能被派生类继承使用，但基类分配器仅优化`sizeof(Base)`对象。若用于分配派生类对象（`sizeof(Derived) > sizeof(Base)`），会导致内存不足。
   - **解法**：检查请求大小，不匹配时转交全局`operator new`：
     ```cpp
     void* Base::operator new(std::size_t size) throw(std::bad_alloc) {
         if (size != sizeof(Base)) 
             return ::operator new(size); // 非Base对象转交全局处理
         // 否则执行Base专属分配逻辑
     }
     ```

#### 3. **异常安全与new-handler机制**

   - **循环尝试分配**：在循环中尝试分配，每次失败后调用当前`new-handler`函数（可能释放备用内存）。
   - **new-handler处理**：
     ```cpp
     while (true) {
         void* ptr = customAlloc(size); // 自定义分配逻辑
         if (ptr) return ptr;
         std::new_handler globalHandler = std::set_new_handler(nullptr);
         std::set_new_handler(globalHandler);
         if (globalHandler) (*globalHandler)(); // 调用处理函数
         else throw std::bad_alloc();
     }
     ```

---

### ⚠️ **二、`operator delete`的实现规范**

#### 1. **安全处理空指针**

   - **标准要求**：删除`nullptr`必须安全无副作用。
   - **实现**：显式检查并跳过：
     ```cpp
     void operator delete(void* rawMemory) noexcept {
         if (rawMemory == nullptr) return; // 空指针直接返回
         customDealloc(rawMemory); // 执行释放逻辑
     }
     ```

#### 2. **处理类专属版本的大小不匹配**

   - **派生类删除基类指针**：若基类未声明虚析构函数，传递给`operator delete`的`size`参数可能错误（因对象大小计算依赖虚表）。
   - **安全措施**：
     ```cpp
     void Base::operator delete(void* rawMemory, std::size_t size) noexcept {
         if (rawMemory == nullptr) return;
         if (size != sizeof(Base)) {
             ::operator delete(rawMemory); // 大小错误转交全局delete
             return;
         }
         customDealloc(rawMemory); // 专属释放逻辑
     }
     ```

---

### 🔍 **三、关键陷阱与规避策略**

#### 1. **数组分配的特殊性**

   - **`operator new[]`的size参数**：可能包含额外信息（如元素数量），不能假设`size = n * sizeof(T)`。
   - **应对**：类专属版本需与单一对象版本分离，避免逻辑依赖数组元素大小。

#### 2. **多线程安全性**

   - **全局状态竞争**：自定义内存管理器需考虑线程安全（如使用原子操作保护分配队列）。
   - **类专属版本**：静态成员变量需加锁或使用`thread_local`。

#### 3. **与构造/析构的交互**

   - **构造函数异常**：若`operator new`成功但构造函数抛出异常，需自动调用匹配的`operator delete`释放内存（编译器自动插入此逻辑）。

---

### 💎 **四、应用场景与决策建议**

| **场景**           | **是否推荐自定义`new/delete`** | **风险等级** | **替代方案**                |
| ------------------ | ------------------------------ | ------------ | --------------------------- |
| 高频小对象分配     | ✅ 强烈推荐                     | 中           | 类专属分配器 + 内存池       |
| 内存使用统计/调试  | ✅ 推荐                         | 低           | 全局钩子或代理分配器        |
| 硬件要求特殊对齐   | ✅ 必须                         | 高           | `align_val_t`参数（C++17）  |
| 多线程高性能分配   | ✅ 推荐                         | 中           | TLS内存池 + 无锁队列        |
| 第三方库兼容性要求 | ❌ 避免                         | 极高         | 容器定制分配器（Allocator） |

> **最佳实践总结**：
> 1. **优先类专属版本**：避免全局替换，减少冲突风险。
> 2. **严格遵循标准行为**：包括0字节、空指针、大小不匹配等边界情况。
> 3. **与RAII结合**：在资源管理类（如智能指针）中封装自定义内存管理，而非直接暴露`new/delete`。
> 4. **性能测试**：自定义分配器需通过性能压测验证优化效果，避免引入新瓶颈。

---

### ⚡ **五、代码示例：带统计的类专属分配器**

```cpp
class MemoryTrackedObject {
public:
    static void* operator new(std::size_t size) {
        if (size != sizeof(MemoryTrackedObject))
            return ::operator new(size);
        totalAllocations += size;
        void* ptr = malloc(size);
        if (!ptr) throw std::bad_alloc();
        return ptr;
    }

    static void operator delete(void* ptr, std::size_t size) noexcept {
        if (!ptr) return;
        if (size != sizeof(MemoryTrackedObject)) {
            ::operator delete(ptr);
            return;
        }
        totalDeallocations += size;
        free(ptr);
    }
private:
    static std::atomic<size_t> totalAllocations;
    static std::atomic<size_t> totalDeallocations;
};
```

此实现满足条款51所有核心要求：
- 处理0字节（通过全局`new`间接支持）
- 检查大小不匹配（转交全局版本）
- 线程安全（`std::atomic`计数）
- 空指针安全（显式检查）

Scott Meyers强调：**自定义`new/delete`是“最接近金属”的C++特性，其正确性直接影响程序根基**。理解并遵守上述规则，方能在性能优化与系统稳定性间取得平衡。
