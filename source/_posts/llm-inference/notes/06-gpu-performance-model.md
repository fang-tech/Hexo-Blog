---
title: 06 · GPU 性能模型与分析工具
type: note
topic: llm-inference
created: 2026-03-31
updated: 2026-03-31
status: completed
tags: [llm-inference, cuda, roofline, arithmetic-intensity, memory-bound, compute-bound, profiling]
---

# 06 · GPU 性能模型与分析工具

> **目标**：理解 Roofline Model，能计算 kernel 的算术强度，判断瓶颈在带宽还是算力，知道该往哪个方向优化

---

## 一、两个硬件上限

GPU 的性能受两个硬件上限约束：

```
算力上限：GPU 每秒能做多少次浮点运算（TFLOPS）
带宽上限：GPU 每秒能从显存搬多少数据（TB/s）
```

4090D 的参数：
- 算力峰值：82.6 TFLOPS（FP32）= 82600 GFLOPS
- 显存带宽：1008 GB/s

---

## 二、算术强度

**算术强度（Arithmetic Intensity）**：kernel 的固有属性，由计算量和数据量的比值决定：

```
算术强度 = 总浮点运算次数（FLOP） / 总显存访问字节数（Byte）
```

不需要运行就能算出来，只需数清楚 kernel 做了多少次运算、读写了多少数据。

**Vector Add 的算术强度：**

```
C[i] = A[i] + B[i]，N 个元素

浮点运算：N 次加法 = N FLOP
显存访问：读 A（4N Byte）+ 读 B（4N Byte）+ 写 C（4N Byte）= 12N Byte

算术强度 = N / 12N = 1/12 ≈ 0.083 FLOP/Byte
```

**Naive GEMM 的算术强度：**

```
C = A × B，N×N 矩阵

浮点运算：2N³ FLOP（每个元素 N 次乘加，共 N² 个元素）
显存访问：每个元素读 A 一行（N 次）+ B 一列（N 次）+ 写 C = (2N+1) 次
          共 N²(2N+1) × 4 Byte ≈ 8N³ Byte

算术强度 = 2N³ / 8N³ = 0.25 FLOP/Byte
```

**规律：计算越密集、数据复用越多，算术强度越高。**

Tiled GEMM 每个 tile 只从 HBM 读一次但被 TILE_SIZE 个 thread 复用，HBM 访问量缩小 TILE_SIZE 倍，算术强度从 0.25 提升到 8.0。

---

## 三、Ridge Point

**Ridge Point** = 算力上限 / 带宽上限，是硬件的固有属性：

```
ridge point = 82600 GFLOPS / 1008 GB/s ≈ 82 FLOP/Byte
```

物理含义：**带宽刚好能喂饱算力时的算术强度**。

- 算术强度 < ridge point → memory-bound（带宽是瓶颈）
- 算术强度 > ridge point → compute-bound（算力是瓶颈）

Ridge point 是分界线，不随 kernel 变化。

---

## 四、Roofline Model

把两个硬件约束画在同一张图上：

```
性能
(GFLOPS)
  |                    ____________  ← 算力上限（水平线）
  |                   /↑
  |                  / ridge point（交点）
  |                 /
  |                /  ← 斜率 = 带宽（GB/s）
  |               /
  |              /
  |             /
  |            /
  |           /
  |          /
  |         /
  |        /
  |       /
  |      /
  |     /
  |    /
  |   /
  |  /
  | /
  |/
  |______________________________
  0                    算术强度（FLOP/Byte）
```

- **斜线段**（算术强度 < ridge point）：memory-bound。性能 = 算术强度 × 带宽，算术强度越高性能越高。优化方向：减少 HBM 访问，提升数据复用（如 tiling）
- **水平线段**（算术强度 > ridge point）：compute-bound。性能 = 算力上限，继续提升算术强度无收益。优化方向：提高算力利用率（如减少 warp divergence、提升 occupancy）

Kernel 在图上的位置：
- x 坐标 = 算术强度
- y 坐标 = 实际性能
- 落在斜线上 → memory-bound
- 落在水平线上 → compute-bound
- 离上限越远，优化空间越大

---

## 五、实验结果分析（4090D，1024×1024 GEMM）

```
=== 理论算术强度 ===
Naive：0.25 FLOP/Byte   → 深度 memory-bound（远低于 ridge point 82）
Tiled：8.00 FLOP/Byte   → 仍然 memory-bound

=== 实测结果 ===
naive：4500 GFLOPS，算力利用率 5.4%
tiled：5763 GFLOPS，算力利用率 7.0%
Speedup：1.3x
```

**为什么 tiled 只快了 1.3x，而不是理论上的 32x：**

理论算术强度计算假设所有数据都从 HBM 读取。但 4090D 的 L2 Cache 有 72MB，缓存了 naive kernel 的大量重复读取，真实 HBM 访问量远小于理论值。我们**高估了 naive 的 HBM 访问字节数**，导致算出来的算术强度（0.25）被低估，实际算术强度比 0.25 高得多，naive 的实际性能因此比理论预测高。Tiled 的优势被硬件缓存抵消，只剩 1.3x。

---

## 六、三个关键 Profile 指标

在有权限的环境下（非容器），用 `ncu` 可以直接读取：

| 指标 | 含义 | 判断方式 |
|------|------|---------|
| Compute Throughput | 实际 GFLOPS / 峰值 GFLOPS | 高 → compute-bound |
| Memory Throughput | 实际带宽 / 峰值带宽 | 高 → memory-bound |
| Occupancy | 实际 warp 数 / 最大 warp 数 | 低 → 延迟隐藏不足 |

两个都低 → occupancy 不足，或存在同步/分支等其他瓶颈。

AutoDL 容器环境不支持 `ncu`（GPU performance counter 被禁用），可用手动计时 + 理论分析替代。

---

## 七、进一步优化 GEMM 的方向

当前 tiled GEMM 算术强度 8.0，距离 ridge point 82 还差 10 倍，三个优化方向：

| 方向 | 原理 | 收益 |
|------|------|------|
| 增大 tile 尺寸 | 更多 thread 复用同一份数据 | 中，受 Shared Memory 容量限制 |
| Register Tiling | 每个 thread 计算多个输出元素，数据复用在寄存器里 | 高，cuBLAS 的核心手段 |
| 向量化访存（float4） | 一条指令读 16 Byte，减少内存事务数 | 中 |

cuBLAS 综合使用以上手段，能达到算力峰值的 80-90%。

---

## 参考材料

1. **NVIDIA Nsight Compute 文档**：https://docs.nvidia.com/nsight-compute/
2. **Roofline Model 原论文**：https://crd.lbl.gov/assets/pubs_presos/parlab08-roofline-talk.pdf
