---
layout: post
title: Effective C++ 条款16：成对使用new和delete时要采取相同形式
categories: [阅读笔记, Effective C++]
tag: [Effective C++]
date: '2025-06-30 00:08:03 +0800'
---

## **Effective C++ 条款16 ：成对使用new和delete时要采取相同形式**

---

<br/>

> **严格匹配`new`/`delete`与`new[]`/`delete[]`的使用形式**，否则将引发**未定义行为**。
{: .prompt-danger}


### ⚠️ 一、核心问题：内存布局的差异

1. **单一对象的内存布局**  
   - 仅包含对象本身数据，无额外信息。  
   - **示例**：`new std::string` 分配一个`string`对象的内存。  
2. **数组对象的内存布局**  
   - 在首元素前存储**数组长度**（编译器隐式添加）。  
   - **示例**：`new std::string[10]` 分配的内存结构： 
 
```cpp
| 数组长度n | string1 | string2 | ... | string10 |
```

---

### 🔧 二、`new`/`delete`的内部机制

1. **`new`的操作步骤**  
   - 分配内存（`operator new`）。  
   - 调用构造函数：  
     - 单一对象：调用**1次**构造函数。  
     - 数组：调用**n次**构造函数（每个元素）。  
2. **`delete`的操作步骤**  
   - 调用析构函数：  
     - `delete`：预期调用**1次**析构函数。  
     - `delete[]`：根据内存中的长度值`n`调用**n次**析构函数。  
   - 释放内存（`operator delete`）。  

---

### 💥 三、不匹配使用的后果（未定义行为）

#### **1. 对单一对象使用 `delete[]`**
   - **问题**：`delete[]`会读取对象前的内存作为数组长度`n`（实际是无效数据），尝试调用`n`次析构函数。  
   - **后果**：  
     - 析构函数被错误调用多次（访问非法内存）。  
     - 程序崩溃或数据损坏。  
   - **示例代码**：  
      ```cpp
      class Demo {
      public:
          Demo()  { std::cout << "构造\n"; }
          ~Demo() { std::cout << "析构\n"; }
      };
          
      int main() {
          Demo* p = new Demo;  // 单一对象
          delete[] p;          // 错误！触发未定义行为
          return 0;
      }
      ```

   - **输出**： 
      ```
      构造
      析构
      析构
      析构...（持续崩溃）
      ```

#### **2. 对数组使用 `delete`**
   - **问题**：`delete`只调用**1次**析构函数（首元素），剩余元素未被销毁。  
   - **后果**：  
     - **内存泄漏**（剩余元素资源未释放）。  
     - 若对象持有外部资源（如文件句柄），资源泄漏。  
   - **示例代码**：  
      ```cpp
      Demo* pArr = new Demo[5];  // 5个对象的数组
      delete pArr;                // 错误！仅析构第一个元素
      ```

   - **输出**： 
      ```
      构造（5次）
      析构（仅1次）  // 剩余4个对象泄漏！
      ```

---

### ⚠️ 四、易错场景与解决方案
#### **1. `typedef` 导致的混淆**
   - **问题**：`typedef`隐藏数组类型，易误用`delete`。  
   - **示例**：  
     ```cpp
     typedef std::string AddressLines[4]; // 数组类型别名
     std::string* pal = new AddressLines; // 实际是 new string[4]
     delete pal;    // 错误！应用 delete[] pal;
     ```
   - **解决**：  
     - **避免对数组使用`typedef`**，改用`std::vector`或`std::array`。

#### **2. 智能指针的局限性**
   - **`std::shared_ptr`/`std::unique_ptr`**：  
     - 析构时默认用`delete`，**不支持数组**（`delete[]`）。  
     - **错误用法**：  
       ```cpp
       std::shared_ptr<int> sp(new int[10]); // 析构时调用delete，错误！
       ```
   - **正确替代**：  
     - 使用`std::vector`或`std::array`容器。  
     - C++17后可用`std::make_unique<T[]>`（需显式指定数组）：  
       ```cpp
       auto p = std::make_unique<Demo[]>(5); // 正确管理数组
       ```

---

### 💎 五、最佳实践总结

| **场景**            | **正确操作**               | **错误操作**              |
| ------------------- | -------------------------- | ------------------------- |
| 单一对象            | `new T; delete;`           | `delete[]`                |
| 对象数组            | `new T[n]; delete[];`      | `delete`                  |
| `typedef`定义的数组 | 改用`std::vector`          | 依赖`typedef`类型         |
| 智能指针管理数组    | `std::make_unique<T[]>(n)` | `shared_ptr<T>(new T[n])` |

1. **严格匹配原则**：  
   - `new` ↔ `delete`  
   - `new[]` ↔ `delete[]`  
2. **优先使用容器**：  
   - 用`std::vector`/`std::array`替代原始数组，避免手动管理。  
3. **警惕智能指针**：  
   - 默认智能指针不管理数组，需显式使用数组特化版本（C++17+）。  

> “C++不提供运行时类型信息来区分`new`的形式，匹配责任完全在程序员肩上。” —— Scott Meyers

---

通过遵循条款16，可彻底避免因内存布局误解导致的内存泄漏、资源泄漏及程序崩溃，是C++资源管理的基石之一。
