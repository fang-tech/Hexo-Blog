---
title: 04 · CUDA 执行模型
type: note
topic: llm-inference
created: 2026-03-31
updated: 2026-03-31
status: completed
tags: [llm-inference, cuda, gpu, thread, warp, block, grid, occupancy, memory-latency]
---

# 04 · CUDA 执行模型

> **目标**：理解 GPU 和 CPU 的执行模型根本差异，掌握 thread/warp/block/grid 四层层级，理解 warp divergence 和延迟隐藏机制

---

## 一、GPU 和 CPU 的根本差异

**高层抽象：**

```
CPU：少量强大的核心（8-32），每个核心有复杂的分支预测、乱序执行、大缓存
     → 设计目标：让单个任务跑得尽可能快

GPU：大量简单的核心（3080 Ti 有 10240 个 CUDA core），每个核心只做简单的乘加
     → 设计目标：让海量任务同时跑
```

矩阵乘法天然适合 GPU——拆成几万个独立的乘加操作，同时计算。

**为什么 GPU 核心可以这么多：**

每个 GPU 核心没有复杂的控制逻辑（分支预测、乱序执行），省下来的晶体管全部用来堆计算单元。代价是编程模型受限——程序员必须显式组织并行。

---

## 二、四层执行层级

### Thread（线程）

最小执行单元，执行一次 kernel 函数调用。每个 thread 有独立的寄存器和局部变量。

例：矩阵加法 `C[i] = A[i] + B[i]`，每个 thread 负责计算一个元素。

### Warp（线程束）

**32 个 thread 组成一个 warp，是 GPU 硬件的调度单位。**

GPU 不单独调度每个 thread，而是以 warp 为单位，32 个 thread 同时执行同一条指令。这叫 **SIMT（Single Instruction Multiple Threads）**。

**为什么以 warp 为单位而不是单个 thread：**

硬件成本问题。给每个 thread 配独立的指令译码器和调度逻辑代价极高。warp 设计让 32 个 thread 共享一套控制硬件，控制硬件成本降低 32 倍，省下的晶体管用来堆更多 CUDA core。代价是 32 个 thread 必须走同一条指令流。

### Warp Divergence（线程束分歧）

同一个 warp 里的 thread 遇到 `if/else` 走不同分支时发生：

```cpp
// 同一 warp 里一半走 if，一半走 else → divergence
if (threadIdx.x % 2 == 0) {
    // 偶数 thread 的计算
} else {
    // 奇数 thread 的计算
}
```

**硬件实现机制：active mask（活跃掩码）**

每个 warp 有一个 32 位的 active mask，记录哪些 thread 当前活跃：

```
第一轮：active mask = 1111...0000（前16个thread执行if，后16个被屏蔽）
        被屏蔽的 thread 照样走指令流，但写入结果被丢弃
第二轮：active mask = 0000...1111（后16个thread执行else，前16个被屏蔽）
```

实际执行时间 = if 分支时间 + else 分支时间，并行度减半。这是硬件层面的串行，不是软件层面的。

### Block（线程块）

多个 thread 组成一个 block。block 内的 thread 可以：
- 通过 **shared memory** 共享数据
- 通过 `__syncthreads()` 同步

block 内最多 1024 个 thread。blockSize 应该是 32 的倍数——如果写 `blockSize = 100`，GPU 会分配 4 个 warp（128 个 thread slot），其中 28 个 slot 是空的，浪费了硬件资源。

**Block 和 Warp 的关系：**

block 是编程模型上的概念，warp 是硬件调度单位。你定义 `blockSize = 256`，GPU 把这 256 个 thread 自动分成 8 个 warp（256 / 32 = 8），以 warp 为单位调度执行：

```
Block（程序员定义）
├── Warp 0（thread 0-31）   ← 硬件实际调度
├── Warp 1（thread 32-63）
├── Warp 2（thread 64-95）
└── ...（共 8 个 warp）
```

**Block 的生命周期：**

```
kernel 启动 → 所有 block 进入等待队列
    ↓
SM 空闲时，从队列取一个 block 开始执行
    ↓
block 内所有 thread 执行完 kernel 函数的最后一行
    ↓
block 结束，SM 释放，Shared Memory 清空
    ↓
SM 取下一个 block
```

`__shared__` 变量的生命周期 = block 的生命周期。block 开始时分配，block 结束时释放。不同 block 的 Shared Memory 互相独立，互不可见。

### Grid（网格）

多个 block 组成一个 grid，是一次 kernel 启动的全部 thread。block 之间不能直接通信，不能保证执行顺序。

```
Grid
├── Block(0,0)  Block(0,1)  Block(0,2)
├── Block(1,0)  Block(1,1)  Block(1,2)
└── Block(2,0)  Block(2,1)  Block(2,2)
        每个 Block 内有多个 Thread
        每 32 个 Thread 组成一个 Warp
```

---

## 三、内置变量与坐标系

CUDA 提供四个内置变量，每个 thread 启动时由 GPU 硬件自动注入：

```
threadIdx：当前 thread 在 block 内的坐标（x, y, z）
blockIdx： 当前 block 在 grid 内的坐标（x, y, z）
blockDim： 每个 block 的尺寸，即有多少个 thread（x, y, z）
gridDim：  grid 的尺寸，即有多少个 block（x, y, z）
```

x/y/z 是三个维度的坐标轴名称，没有特殊含义：
- 一维：处理数组 → 只用 x
- 二维：处理矩阵 → 用 x 和 y
- 三维：处理三维数据 → 用 x、y、z

**坐标与 block 是一一对应的关系。** grid 的形状在 kernel 启动时固定，`(0,0)` 和 `(1,1)` 在任何情况下都是两个不同的 block，不存在同一个 block 有多种坐标的情况。

## 四、Thread 下标计算

**一维（vector add）：**

```cpp
int i = blockIdx.x * blockDim.x + threadIdx.x;
// 本质：block 的起始位置 + thread 在 block 内的偏移
```

具体数字（blockSize=256）：

```
Block 0 的 thread 0：  i = 0 * 256 + 0   = 0
Block 0 的 thread 255：i = 0 * 256 + 255  = 255
Block 1 的 thread 0：  i = 1 * 256 + 0   = 256
Block 1 的 thread 1：  i = 1 * 256 + 1   = 257
```

**二维（GEMM）：**

矩阵是二维的，自然地用二维坐标映射。x 对应列，y 对应行：

```cpp
int row = blockIdx.y * blockDim.y + threadIdx.y;  // 负责 C 的第几行
int col = blockIdx.x * blockDim.x + threadIdx.x;  // 负责 C 的第几列
```

假设矩阵 4×4，TILE_SIZE=2，block(1,1) 里的 thread(0,1)：

```
row = blockIdx.y * TILE_SIZE + threadIdx.y = 1 * 2 + 1 = 3
col = blockIdx.x * TILE_SIZE + threadIdx.x = 1 * 2 + 0 = 2
→ 负责计算 C[3][2]
```

和一维完全相同的逻辑，x 和 y 两个方向独立计算。

---

## 四、CPU / GPU 内存模型与数据传输

CPU 和 GPU 是两块独立硬件，各自有独立的内存：

```
CPU ←→ RAM（几十 GB）
GPU ←→ VRAM/显存（12 GB）
两者通过 PCIe 总线连接，带宽约 16 GB/s
```

GPU 不能直接读 CPU 的内存，必须先用 `cudaMemcpy` 复制到显存：

```
h_A（CPU内存）─── cudaMemcpy HostToDevice ──→ d_A（GPU显存）
                        kernel 计算
h_C（CPU内存）←── cudaMemcpy DeviceToHost ─── d_C（GPU显存）
```

`h_` 前缀 = host（CPU），`d_` 前缀 = device（GPU）——CUDA 代码的命名惯例。

PCIe 传输是推理延迟的重要来源，推理优化要尽量减少 CPU/GPU 之间的数据搬运。

---

## 五、Occupancy 与延迟隐藏

**SM（Streaming Multiprocessor）** 是 GPU 的计算单元，3080 Ti 有 80 个 SM。每个 SM 可以同时持有多个 warp。

**延迟隐藏机制：**

显存访问延迟约 200-800 个时钟周期。单个 warp 等待内存时，SM 切换到其他 warp 继续执行，让多个 warp 的内存请求同时在飞：

```
时间轴：
Warp A：发出内存请求 ──等待──→ 拿到数据，计算
Warp B：        发出内存请求 ──等待──→ 拿到数据，计算
Warp C：                发出内存请求 ──等待──→ 拿到数据，计算
```

三个 warp 的等待时间重叠，内存控制器并行处理所有请求。只有一个 warp 时，等待时间完全浪费。

**Occupancy（占用率）**：SM 实际运行的 warp 数 / SM 最大支持的 warp 数。Occupancy 越高，延迟隐藏效果越好。

**实验结果（16M 元素 vector add，3080 Ti）：**

```
blockSize=  32  time=0.378 ms   ← occupancy 低，延迟隐藏差
blockSize=  64  time=0.243 ms   ← occupancy 提升，延迟被充分隐藏
blockSize= 128  time=0.243 ms
blockSize= 256  time=0.244 ms   ← 64 之后持平：瓶颈转移到显存带宽
blockSize= 512  time=0.244 ms
blockSize=1024  time=0.245 ms
```

64 之后持平的原因：这是 memory-bound kernel（只做一次加法，主要时间在读写显存）。occupancy 够了之后，瓶颈是显存带宽上限，再增大 blockSize 不会让显存更快。

---

## 六、第一个 Kernel：Vector Add

```cuda
__global__ void vector_add(float* A, float* B, float* C, int N) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < N) C[i] = A[i] + B[i];
}

// 启动方式
int blockSize = 256;
int gridSize = (N + blockSize - 1) / blockSize;  // ceil(N / blockSize)
vector_add<<<gridSize, blockSize>>>(d_A, d_B, d_C, N);
```

`__global__`：在 GPU 上执行，从 CPU 调用。

`if (i < N)`：防止最后一个 block 的部分 thread 越界（N 不一定是 blockSize 的整数倍）。

---

## 参考材料

1. **CUDA C Programming Guide Ch1-3**：https://docs.nvidia.com/cuda/cuda-c-programming-guide/
