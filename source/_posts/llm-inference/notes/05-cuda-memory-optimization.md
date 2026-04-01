---
title: 05 · CUDA 内存优化
type: note
topic: llm-inference
created: 2026-03-31
updated: 2026-03-31
status: completed
tags: [llm-inference, cuda, shared-memory, tiling, gemm, bank-conflict, memory-hierarchy]
---

# 05 · CUDA 内存优化

> **目标**：理解 GPU 内存层级，掌握 Shared Memory Tiling 优化 GEMM 的原理，理解 bank conflict 和延迟隐藏机制

---

## 一、GPU 内存层级

**高层抽象：**

```
寄存器（Register）  →  Shared Memory  →  L2 Cache  →  HBM（显存）
     最快                                                  最慢
     最小                                                  最大
```

| 层级 | 作用域 | 延迟 | 容量（3080 Ti） |
|------|--------|------|----------------|
| 寄存器 | 每个 thread 私有 | ~0 | 极小 |
| Shared Memory | block 内共享 | ~5ns | 每个 SM 约 100KB |
| L2 Cache | 所有 SM 共享 | 中等 | 6MB（4090D 72MB）|
| HBM | 所有 SM 共享 | 200-800 时钟周期 | 12GB |

**核心推论：** 同一时刻需要被多次读取的数据，应该先从 HBM 搬到 Shared Memory，再从 Shared Memory 读取计算——减少 HBM IO 次数。

---

## 二、CUDA 编程的固定骨架

所有 CUDA 程序围绕一个核心问题：CPU 和 GPU 是两块独立硬件，内存独立，不能直接互访。

```
1. 在 GPU 上分配内存（cudaMalloc）
2. 把数据从 CPU 内存复制到 GPU 内存（cudaMemcpy HostToDevice）
3. 启动 kernel，在 GPU 上并行计算（<<<grid, block>>>）
4. 把结果从 GPU 内存复制回 CPU 内存（cudaMemcpy DeviceToHost）
5. 释放 GPU 内存（cudaFree）
```

关键语法逐行说明：

```cuda
float *d_A;
cudaMalloc(&d_A, size);
// cudaMalloc：在 GPU 显存分配内存，返回的指针只能在 GPU 上用
// &d_A：传指针的地址，cudaMalloc 把分配好的显存地址写入 d_A

cudaMemcpy(d_A, h_A, size, cudaMemcpyHostToDevice);
// 把数据从 CPU 内存复制到 GPU 显存
// 方向：cudaMemcpyHostToDevice（CPU→GPU）或 cudaMemcpyDeviceToHost（GPU→CPU）

vector_add<<<gridSize, blockSize>>>(d_A, d_B, d_C, N);
// <<<gridSize, blockSize>>>：kernel 启动语法
// kernel 是异步的，CPU 调用后不等 GPU 算完就继续执行

cudaMemcpy(h_C, d_C, size, cudaMemcpyDeviceToHost);
// 隐式等待 GPU 完成，再把结果复制回 CPU

cudaFree(d_A);
// 释放 GPU 显存，对应 cudaMalloc，不释放会泄漏显存
```

`__global__`：kernel 函数的标志，在 GPU 上执行，从 CPU 调用。函数参数里的指针必须是 GPU 显存指针，传 CPU 内存指针会崩溃。

`__shared__`：声明 Shared Memory 变量，block 内所有 thread 共享，生命周期等于 block 的生命周期。

## 三、GEMM 是什么

GEMM（General Matrix Multiply）：`C = A × B`，A 是 M×K，B 是 K×N，C 是 M×N。

推理计算量最大的操作全是 GEMM：
- FFN 的 W₁、W₂ 矩阵乘法
- Attention 的 QKᵀ 点积
- Attention 的 score × V

---

## 三、Naive GEMM 的问题

每个 thread 计算 C 的一个元素，直接从 HBM 读数据：

```cuda
__global__ void gemm_naive(float* A, float* B, float* C, int n) {
    int row = blockIdx.y * blockDim.y + threadIdx.y;
    int col = blockIdx.x * blockDim.x + threadIdx.x;
    if (row < n && col < n) {
        float sum = 0.0f;
        for (int k = 0; k < n; k++)
            sum += A[row * n + k] * B[k * n + col];
        C[row * n + col] = sum;
    }
}
```

问题：A 的同一行被 N 个 thread 各从 HBM 读一次，B 的同一列被 M 个 thread 各从 HBM 读一次。大量重复的 HBM 读取。

---

## 四、Shared Memory Tiling

**核心思路：** 把 A 和 B 分成 TILE_SIZE × TILE_SIZE 的小块，每次协作把一个 tile 加载到 Shared Memory，block 内所有 thread 复用这份数据。

```
HBM → Shared Memory（第一个 __syncthreads__ 等待这步完成）
Shared Memory → 寄存器 → 计算（第二个 __syncthreads__ 等待这步完成）
```

`__syncthreads()` 的作用：
- **第一个**：等所有 thread 把数据从 HBM 搬到 Shared Memory。如果去掉，某些 thread 还没写完，其他 thread 就开始读，得到垃圾数据
- **第二个**：等所有 thread 计算完毕，再加载下一个 tile，避免覆盖还在使用中的数据

```cuda
__global__ void gemm_tiled(float* A, float* B, float* C, int n) {
    __shared__ float tileA[TILE_SIZE][TILE_SIZE];
    __shared__ float tileB[TILE_SIZE][TILE_SIZE];

    int row = blockIdx.y * TILE_SIZE + threadIdx.y;
    int col = blockIdx.x * TILE_SIZE + threadIdx.x;
    float sum = 0.0f;

    for (int t = 0; t < n / TILE_SIZE; t++) {
        // 协作加载 tile 到 Shared Memory
        tileA[threadIdx.y][threadIdx.x] = A[row * n + (t * TILE_SIZE + threadIdx.x)];
        tileB[threadIdx.y][threadIdx.x] = B[(t * TILE_SIZE + threadIdx.y) * n + col];
        __syncthreads();  // 等所有 thread 加载完

        // 在 Shared Memory 里计算
        for (int k = 0; k < TILE_SIZE; k++)
            sum += tileA[threadIdx.y][k] * tileB[k][threadIdx.x];
        __syncthreads();  // 等所有 thread 算完，再加载下一个 tile
    }

    if (row < n && col < n)
        C[row * n + col] = sum;
}
```

每个 tile 只从 HBM 读一次，被 block 内 TILE_SIZE 个 thread 复用——HBM 访问减少 TILE_SIZE 倍。

---

## 五、实验结果（4090D，1024×1024）

```
naive    time=0.477 ms  GFLOPS=4498.9
tiled    time=0.373 ms  GFLOPS=5755.1
Speedup: 1.3x
```

4090D 的 L2 Cache 是 72MB，几乎把整个 1024×1024 矩阵（4MB）缓存住了。naive kernel 的重复 HBM 读取大部分命中 L2，tiling 的优势被硬件缓存抵消。

两个版本都只达到 4090D 理论峰值（82.6 TFLOPS）的 5-7%——完全是 memory-bound，算力没有被用满。

**优化效果取决于硬件：** 在 L2 更小的老 GPU 上，tiling 的提升可以达到 5-10x。理解内存层级和数据复用是写高性能 kernel 的基础，不因为硬件自动缓存了就失去意义。

---

## 六、Bank Conflict

Shared Memory 被分成 32 个 bank，每个 bank 宽度 4 字节。同一个 warp 里的 thread 访问 Shared Memory 时，如果多个 thread 访问同一个 bank 的不同地址，会被串行化。

```
无冲突（每个 thread 访问不同 bank）：
thread 0 → bank 0，thread 1 → bank 1，...  → 一次完成

有冲突（多个 thread 访问同一 bank）：
thread 0 → bank 0，thread 1 → bank 0  → 串行执行，性能减半
```

**tiled kernel 里的问题：**

内循环 `tileA[threadIdx.y][k]` 中，所有 thread 的 k 相同，threadIdx.y 不同，访问的是同一列（列优先访问），间隔 TILE_SIZE 个元素，导致 bank conflict。

**优化方式：加载时转置存储 tileA**

```cuda
// 加载时转置
tileA[threadIdx.x][threadIdx.y] = A[row * n + (t * TILE_SIZE + threadIdx.x)];

// 内循环变成行访问，连续，无 bank conflict
sum += tileA[k][threadIdx.y] * tileB[k][threadIdx.x];
```

---

## 七、Warp 调度与指令执行

同一个 warp 内的 32 个 thread 在同一时刻执行同一条指令（SIMT）。

不同 warp 之间完全独立，各自执行各自的指令：

```
Warp 0：执行 load A[i]
Warp 1：执行 multiply
Warp 2：执行 store C[i]
        ← 三个 warp 同时在 SM 上，执行不同指令
```

SM 的调度器每个时钟周期选一个 ready 的 warp 发射指令，在多个 warp 之间切换，让执行单元始终有活干——这是延迟隐藏的底层机制。

---

## 参考材料

1. **CUDA C Programming Guide Ch5-6**：https://docs.nvidia.com/cuda/cuda-c-programming-guide/
2. **how-to-optimize-gemm**：https://github.com/yzhaiustc/Optimizing-GEMM-on-NVIDIA-Turing-GPUs
