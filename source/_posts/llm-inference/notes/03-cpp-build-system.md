---
title: 03 · C++ 编译模型与 CMake
type: note
topic: llm-inference
created: 2026-03-30
updated: 2026-03-30
status: completed
tags: [llm-inference, cpp, cmake, compilation, linking, header, preprocessor]
---

# 03 · C++ 编译模型与 CMake

> **目标**：理解 C++ 从源文件到可执行文件的完整流程，以及 .h/.cpp 分离的原因，能独立用 CMake 构建多文件项目

---

## 一、编译流程总览

```
源文件 (.cpp)
    ↓ 预处理器
翻译单元（纯文本）
    ↓ 编译器
目标文件 (.o)
    ↓ 链接器
可执行文件
```

每个 `.cpp` 文件独立编译成一个 `.o`，链接器最后把所有 `.o` 合并。

---

## 二、第一步：预处理器

**输入**：原始 `.cpp` 源文件（含 `#include`、`#define` 等指令）

**做了什么**：

1. **展开 `#include`**：把头文件内容直接复制粘贴到 `#include` 的位置

```cpp
// 原始 main.cpp
#include "add.h"
int main() { ... }

// 展开后
int add(int a, int b);   // add.h 的内容被复制进来
int main() { ... }
```

`#include <iostream>` 同理，展开后有几千行。

2. **展开 `#define`**：文本替换

```cpp
#define MAX_SIZE 1024
int arr[MAX_SIZE];
// 展开后
int arr[1024];
```

3. **处理条件编译**：`#ifdef` / `#ifndef` 等，不满足条件的代码块直接删掉

**输出**：**翻译单元**——所有 `#` 指令处理完之后的纯 C++ 文本，没有任何 `#` 开头的指令。可以用 `g++ -E main.cpp` 查看。

---

## 三、第二步：编译器

**输入**：翻译单元（纯 C++ 文本）

**做了什么**：词法分析 → 语法分析（生成 AST）→ 语义分析（类型检查）→ 生成机器码

关键点：编译器只看当前翻译单元，遇到 `add(1, 2)` 的调用时：

```cpp
// main.cpp 展开后
int add(int a, int b);   // 只有声明，知道函数签名
int main() {
    add(1, 2);           // 生成调用代码，但地址不知道
}
```

编译器从声明知道参数类型和返回类型，可以生成正确的调用代码，但 `add` 的实现在哪里它不知道——在 `.o` 文件里留一个**未解析符号**，标记为 `U`（undefined）。

**输出**：`.o` 目标文件，包含：
- 机器码
- **符号表**：已定义的符号（`T`，有实现）和未解析的符号（`U`，地址待定）

```bash
nm main.o
# U add    ← undefined，地址待定
# T main   ← defined，地址已知
```

---

## 四、第三步：链接器

**输入**：所有 `.o` 文件

```
main.o  →  机器码 + 未解析符号：add（U）
add.o   →  机器码 + 已定义符号：add（T）
```

**做了什么**：

1. 合并所有 `.o` 的机器码，分配最终地址
2. 解析未解析符号：`main.o` 里 `add` 地址待定 → 在 `add.o` 里找到实现 → 填入真实地址

**两种链接错误：**

- `undefined reference to 'add'`：有声明和调用，但所有 `.o` 里都找不到定义
- `multiple definition of 'add'`：多个 `.o` 里都有同名定义，不知道用哪个

**输出**：可执行文件，所有地址填好，可以直接运行

---

## 五、.h 和 .cpp 分离的原因

**声明 vs 定义：**

```cpp
int add(int a, int b);                   // 声明：描述函数签名，不含实现，可出现多次
int add(int a, int b) { return a + b; }  // 定义：包含实现，只能出现一次
```

分离的做法：

```cpp
// add.h：只放声明，被多个文件 #include，每次预处理展开一次，没有重复定义问题
int add(int a, int b);

// add.cpp：只放定义，只编译一次，生成唯一的 add.o
#include "add.h"
int add(int a, int b) { return a + b; }
```

数据流：

```
add.h ──────────────────────────────────────┐
                                             ↓ #include 展开
main.cpp ──→ 预处理器 ──→ 翻译单元 ──→ 编译器 ──→ main.o（add: U）
                                                        ↓
add.cpp ──→ 预处理器 ──→ 翻译单元 ──→ 编译器 ──→ add.o（add: T）──→ 链接器 ──→ my_program
```

`#include "add.h"` vs `#include <iostream>`：
- 双引号：先在当前目录找，找不到再找系统路径
- 尖括号：只找系统路径（标准库、已安装的第三方库）

---

## 六、CMake

手动编译多文件项目繁琐，CMake 通过 `CMakeLists.txt` 描述项目结构，自动生成编译命令。

**最小项目结构：**

```
my_project/
├── CMakeLists.txt
├── add.h
├── add.cpp
└── main.cpp
```

**CMakeLists.txt：**

```cmake
cmake_minimum_required(VERSION 3.10)
project(my_project)
set(CMAKE_CXX_STANDARD 17)
add_executable(my_program main.cpp add.cpp)
```

`add_executable(my_program main.cpp add.cpp)` 等价于：

```bash
g++ -c main.cpp -o main.o
g++ -c add.cpp  -o add.o
g++ main.o add.o -o my_program
```

漏掉 `add.cpp`：链接器报 `undefined reference to 'add'`。

**构建命令：**

```bash
mkdir build && cd build
cmake ..     # 读取 CMakeLists.txt，生成 Makefile
make         # 执行编译和链接
./my_program
```

**工业项目的组织方式：** 不把所有文件列入 `add_executable`，而是用库来组织：

```cmake
add_library(my_lib add.cpp utils.cpp)
add_executable(my_program main.cpp)
target_link_libraries(my_program my_lib)
```

大项目每个子目录有自己的 `CMakeLists.txt`，各自打包成库，顶层用 `add_subdirectory` 组合。

---

## 参考材料

1. **learncpp.com（编译流程）**：https://www.learncpp.com/cpp-tutorial/introduction-to-the-preprocessor/
2. **CMake Tutorial**：https://cmake.org/cmake/help/latest/guide/tutorial/
