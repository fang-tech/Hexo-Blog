---
title: 02 · C++ 必要基础
type: note
topic: llm-inference
created: 2026-03-30
updated: 2026-03-30
status: completed
tags: [llm-inference, cpp, memory, raii, smart-pointer, pointer, reference, template, move-semantics]
---

# 02 · C++ 必要基础

> **目标**：理解 C++ 的内存模型和对象生命周期管理，能看懂推理框架 host 代码里的指针、智能指针、模板用法

---

## 一、指针与引用

C++ 在 C 的指针基础上增加了引用：

```cpp
int x = 42;
int* p = &x;   // 指针：存储 x 的地址
int& r = x;    // 引用：x 的别名，本质是语法糖，编译器实现上通常也是地址
```

| | 指针 | 引用 |
|--|------|------|
| 是否可以为 null | 可以 | 不行，必须绑定到对象 |
| 是否可以改变指向 | 可以 | 不行，绑定后不能换 |
| 语法 | `*p` 解引用 | 直接用 `r`，和用原变量一样 |

**函数传参的三种方式：**

```cpp
void foo_value(int x)   { x = 99; }   // 复制，不影响外部
void foo_ptr(int* x)    { *x = 99; }  // 传地址，可修改外部
void foo_ref(int& x)    { x = 99; }   // 传引用，可修改外部，语法更干净
```

推理框架里大量用 `const` 引用传大对象，避免复制开销：

```cpp
void process(const std::vector<float>& tensor) { ... }
// 不复制 tensor，只传引用；const 保证函数内不能修改
```

---

## 二、构造函数与析构函数

对象创建时调用构造函数，销毁时调用析构函数：

```cpp
class Buffer {
public:
    Buffer(int size) {   // 构造函数：对象创建时调用
        data = new float[size];
    }
    ~Buffer() {          // 析构函数：对象销毁时调用
        delete[] data;
    }
private:
    float* data;
};
```

析构函数的调用时机：
- 栈上对象：离开作用域时（`}` 处）自动调用，包括提前 `return` 的情况
- 堆上对象：`delete` 时调用，或智能指针析构时调用

---

## 三、RAII 与智能指针

**RAII（Resource Acquisition Is Initialization）**：把资源的生命周期绑定到对象的生命周期。构造时获取资源，析构时释放资源，编译器保证析构一定被调用。

C 的问题：

```c
int* p = (int*)malloc(sizeof(int));
// 中间可能提前 return → free 被跳过 → 内存泄漏
free(p);
```

C++ 用智能指针解决堆上对象的生命周期管理：

```cpp
auto p = std::make_unique<int>(42);
// 不管怎么 return，离开作用域时析构函数自动调用 delete
```

**什么时候用堆（智能指针），什么时候用栈（局部变量）：**

| 情况 | 选择 |
|------|------|
| 小对象，生命周期在当前作用域内 | 栈，直接声明局部变量 |
| 对象太大（大 tensor buffer、模型权重） | 堆 + 智能指针 |
| 生命周期需要跨函数边界传递 | 堆 + 智能指针 |
| 运行时才知道大小或数量 | 堆 + 智能指针 |

**栈比堆快**：栈分配只是移动 stack pointer，几乎零开销；堆分配需要内存分配器查找空闲块。能用栈就用栈。

**三种智能指针：**

| 类型 | 所有权语义 | 释放时机 |
|------|-----------|---------|
| `unique_ptr` | 独占，不可复制 | 离开作用域时析构 |
| `shared_ptr` | 共享，引用计数 | 引用计数归零时析构 |
| `weak_ptr` | 不持有所有权 | 不触发释放，配合 shared_ptr 打破循环引用 |

---

## 四、移动语义（std::move）

`unique_ptr` 不能复制——复制会导致两个指针都认为自己拥有内存，析构时 double free：

```cpp
auto p1 = std::make_unique<int>(42);
auto p2 = p1;             // 编译错误！
auto p2 = std::move(p1);  // 正确：转移所有权，p1 变为 nullptr
```

`std::move` 不移动任何内存，只是告诉编译器"允许从 p1 窃取资源"。移动后 p1 为空，p2 接管所有权。

推理框架里传递大 tensor buffer 时用移动语义，避免复制：

```cpp
void enqueue(std::unique_ptr<Tensor> tensor) { ... }
enqueue(std::move(my_tensor));  // 转移所有权，不复制数据
```

---

## 五、模板基础

模板让一份代码支持多种类型，在编译期展开，运行时无额外开销：

```cpp
template<typename T>
T add(T a, T b) { return a + b; }

add<int>(1, 2);        // 编译器生成 int 版本
add<float>(1.0f, 2.0f); // 编译器生成 float 版本
```

CUDA kernel 大量用模板支持 FP16/BF16/FP32 等不同精度：

```cpp
template<typename T>
__global__ void gemm_kernel(T* A, T* B, T* C, int N) {
    // 同一份 kernel，支持不同数据类型
}
```

模板在编译期展开，每个具体类型生成一份独立机器码，没有运行时类型判断的开销。

---

## 六、C++ vs Java 内存模型对比

| | C++ | Java |
|--|-----|------|
| 对象可以在栈上吗 | 可以 | 不行，所有对象在堆上 |
| 赋值语义 | 值拷贝（默认复制对象） | 引用传递（复制的是引用） |
| 内存释放 | 析构函数 / 智能指针 | GC 自动回收 |
| 需要关心对象在哪里 | 需要，影响性能和生命周期 | 不需要 |

C++ 最容易踩的坑：

```cpp
MyObject obj1;
MyObject obj2 = obj1;  // 值拷贝！obj2 是独立副本，不是引用
MyObject* ptr = &obj1; // 这才是类似 Java 引用的用法
```

---

## 参考材料

1. **learncpp.com（指针/引用）**：https://www.learncpp.com/cpp-tutorial/introduction-to-pointers/
2. **learncpp.com（智能指针/RAII）**：https://www.learncpp.com/cpp-tutorial/introduction-to-smart-pointers-move-semantics/
3. **learncpp.com（模板）**：https://www.learncpp.com/cpp-tutorial/function-templates/
