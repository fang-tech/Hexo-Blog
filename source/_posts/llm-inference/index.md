---
title: LLM 推理优化学习主干道
type: index
topic: llm-inference
created: 2026-03-29
updated: 2026-03-29
status: in-progress
tags: [llm, inference, cuda, transformer, ai-infra, cpp]
---

# LLM 推理优化学习主干道

## 前置差距分析

> 分析日期：2026-03-29

| 前置能力                                       | 是否具备    | 处理方式            |
| ------------------------------------------ | ------- | --------------- |
| Transformer / Attention 基本概念               | ❌ 缺失    | 纳入主干道节点 01      |
| C++ 基础（指针、内存模型、RAII）                       | ❌ 缺失    | 纳入主干道节点 02-03   |
| CUDA 执行模型（thread/warp/block/shared memory） | ❌ 缺失    | 纳入主干道节点 04-05   |
| GPU 内存层级（HBM/SRAM/带宽瓶颈）                    | ❌ 缺失    | 纳入主干道节点 06      |
| LLM 推理系统（KV cache、batching、调度）             | ❌ 缺失    | 纳入主干道节点 07-11   |
| OS 底层（内存管理、进程、调度）                          | ✅ 已具备   | 节点 06、08 直接复用   |
| 向量化执行引擎经验（Gluten）                          | ⚠️ 部分相关 | 节点 05、09 作为理解锚点 |
| Java / Spark Infra 经验                      | ✅ 已具备   | 节点 11 系统设计时复用   |

---

## 核心抽象

LLM 推理优化是一个 **在有限硬件资源上最大化语言模型服务吞吐与最小化延迟的工程问题**，所有设计围绕一个核心约束展开：

> **GPU 显存有限、带宽有限、算力有限——推理优化的本质是在这三个维度上做最优的取舍。**

从这个约束出发，可以推导出它的所有核心设计：

| 抽象                          | 一句话                                    |
| --------------------------- | -------------------------------------- |
| **Transformer / Attention** | 模型计算的基本单元，推理优化的对象                      |
| **C++ / CUDA**              | 推理系统的实现语言，所有 kernel 的载体                |
| **KV Cache**                | 用显存换计算，避免重复计算历史 token 的 attention      |
| **Prefill / Decode 分离**     | 两个阶段特性不同，需要不同的优化策略                     |
| **Batching**                | 把多个请求打包计算，提升 GPU 利用率                   |
| **Memory Hierarchy**        | SRAM / HBM / DRAM 速度差异决定了 kernel 优化的方向 |
| **Quantization**            | 用精度换显存和速度，是部署侧最常见的压缩手段                 |

---

## 主干道节点

### 第一层：模型与语言基础（不可跳过的前置）

| #   | 文件                                                         | 内容                                     | 状态     |
| --- | ---------------------------------------------------------- | -------------------------------------- | ------ |
| 1   | [01-transformer-basics.md](notes/01-transformer-basics.md) | Transformer 结构与 Attention 计算：模型推理时在算什么 | ✅ 已完成 |
| 2   | [02-cpp-essentials.md](notes/02-cpp-essentials.md)         | C++ 必要基础：指针、内存模型、RAII、智能指针、模板基础        | ✅ 已完成  |
| 3   | [03-cpp-build-system.md](notes/03-cpp-build-system.md)     | C++ 编译模型与 CMake：能独立构建和运行 C++ 项目        | ✅ 已完成  |

### 第二层：GPU 编程基础

| #   | 文件                                                                     | 内容                                                                 | 状态    |
| --- | ---------------------------------------------------------------------- | ------------------------------------------------------------------ | ----- |
| 4   | [04-cuda-execution-model.md](notes/04-cuda-execution-model.md)         | CUDA 执行模型：thread / warp / block / grid + 第一个 kernel                | ✅ 已完成 |
| 5   | [05-cuda-memory-optimization.md](notes/05-cuda-memory-optimization.md) | CUDA 内存优化：coalescing / shared memory / bank conflict / GEMM tiling | ✅ 已完成 |
| 6   | [06-gpu-performance-model.md](notes/06-gpu-performance-model.md)       | GPU 性能模型：HBM/SRAM 层级、roofline、用 nsight 分析瓶颈                        | ✅ 已完成 |

### 第三层：推理系统核心

| #   | 文件                                                                       | 内容                                                  | 状态    |
| --- | ------------------------------------------------------------------------ | --------------------------------------------------- | ----- |
| 7   | [07-autoregressive-generation.md](notes/07-autoregressive-generation.md) | 自回归生成与 KV Cache：token by token 生成机制和显存代价            | ✅ 已完成 |
| 8   | [08-batching-scheduling.md](notes/08-batching-scheduling.md)             | Batching 与调度：static batching vs continuous batching | ✅ 已完成 |
| 9   | [09-paged-attention.md](notes/09-paged-attention.md)                     | PagedAttention 与显存管理：vLLM 的核心设计                     | ⬜ 未开始 |
| 10  | [10-flash-attention.md](notes/10-flash-attention.md)                     | FlashAttention：IO-aware kernel 优化                   | ⬜ 未开始 |
| 11  | [11-quantization.md](notes/11-quantization.md)                           | 量化：用精度换速度和显存（PTQ / INT8 / FP8）                      | ⬜ 未开始 |
| 12  | [12-inference-system-design.md](notes/12-inference-system-design.md)     | 推理系统设计全景：延迟 / 吞吐 / 成本的工程取舍                          | ⬜ 未开始 |

---

## 节点依赖

```
01-transformer-basics（模型基础）
    └── 02-cpp-essentials
            └── 03-cpp-build-system
                    └── 04-cuda-execution-model
                            └── 05-cuda-memory-optimization
                                    └── 06-gpu-performance-model
                                            ├── 07-autoregressive-generation
                                            │       └── 08-batching-scheduling
                                            │               └── 09-paged-attention
                                            └── 10-flash-attention（依赖 05 + 07）
                                                    └── 11-quantization（可并行于 09）
                                                            └── 12-inference-system-design（综合）
```

---

## 学习目标

完成全部节点后，你应该能够：

1. 解释 Transformer attention 的完整计算流程
2. 用 C++ 写基本的系统级代码，读懂推理框架的 host 代码
3. 写出正确的 CUDA kernel（GEMM naive → shared memory tiling）
4. 用 nsight 分析 kernel 的 occupancy 和 memory throughput
5. 解释 KV cache 的作用、显存占用和管理策略
6. 解释 continuous batching 为什么比 static batching 高效
7. 解释 PagedAttention 的设计动机和实现思路
8. 解释 FlashAttention 为什么快（IO-aware）
9. 解释量化的基本原理和常见方案
10. 读懂 vLLM 的 scheduler 逻辑和核心 CUDA kernel

---

## 节点详情与实践计划

### 节点 01 · Transformer 结构与 Attention 计算
- **核心问题**：模型推理时在算什么？矩阵乘法和 Attention 是什么关系？
- **参考材料**：[The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/)、[Attention Is All You Need](https://arxiv.org/abs/1706.03762)
- **实践**：本机用 Python/NumPy 手写 scaled dot-product attention（约 10 行），验证公式
- **完成标准**：能解释 Q/K/V 的角色、attention 计算步骤、为什么除以 √d_k

### 节点 02 · C++ 必要基础
- **核心问题**：C++ 和 Java 的本质差异？指针、栈堆、RAII 解决了什么问题？
- **参考材料**：[learncpp.com](https://www.learncpp.com)（Ch9-12 指针/引用，Ch15-16 RAII/智能指针，Ch26 模板基础）
- **实践**：本机写 3 个小程序：①手动 malloc/free 演示 dangling pointer；②用 unique_ptr 改写；③写一个模板函数
- **完成标准**：能解释为什么 Java 程序员写 C++ 容易出 UB；能用 CMake 构建运行项目

### 节点 03 · C++ 编译模型与 CMake
- **核心问题**：C++ 的编译-链接流程？头文件/源文件分离的意义？
- **参考材料**：[learncpp.com Ch7](https://www.learncpp.com/cpp-tutorial/introduction-to-the-preprocessor/)、[CMake Tutorial](https://cmake.org/cmake/help/latest/guide/tutorial/)
- **实践**：本机用 CMake 构建多文件项目（.h + .cpp + main），`cmake .. && make` 跑通
- **完成标准**：能解释 .h/.cpp 分离的原因；能独立搭建 CMake 项目

### 节点 04 · CUDA 执行模型
- **核心问题**：GPU 和 CPU 的执行模型根本差异？thread/warp/block/grid 如何组织？
- **参考材料**：[CUDA C Programming Guide Ch1-3](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- **环境**：AutoDL RTX 3090（选 CUDA 12.x 镜像，按小时计费，用完关机）
- **实践**：在 AutoDL 上写并运行：①vector add kernel；②改变 block size 观察性能差异
- **完成标准**：能画出 thread/warp/block/grid 层级图；能解释 warp divergence 是什么

### 节点 05 · CUDA 内存优化
- **核心问题**：为什么 naive kernel 慢？coalescing 和 shared memory tiling 如何解决？
- **参考材料**：[CUDA C Programming Guide Ch5-6](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)、[how-to-optimize-gemm](https://github.com/yzhaiustc/Optimizing-GEMM-on-NVIDIA-Turing-GPUs)
- **环境**：AutoDL RTX 3090
- **实践**：实现三版 GEMM：①naive（global memory）；②shared memory tiling；③对比 GFLOPS
- **完成标准**：tiling 版比 naive 快 5x+；能解释为什么

### 节点 06 · GPU 性能模型与分析工具
- **核心问题**：kernel 的瓶颈是算力还是带宽？如何用 roofline 判断？nsight 怎么读？
- **参考材料**：[NVIDIA Nsight Compute 文档](https://docs.nvidia.com/nsight-compute/)
- **环境**：AutoDL RTX 3090（nsight-compute 已预装）
- **实践**：对节点 05 的 GEMM kernel 跑 `ncu` profile，读 occupancy / memory throughput / compute throughput
- **完成标准**：能用 roofline 判断 compute-bound 还是 memory-bound；能解释 occupancy 低的原因

### 节点 07 · 自回归生成与 KV Cache
- **核心问题**：token by token 生成的机制？KV cache 存了什么，为什么能省计算？
- **参考材料**：[vLLM 博客](https://blog.vllm.ai/2023/06/20/vllm.html)、[Lilian Weng: LLM Inference](https://lilianweng.github.io/posts/2023-01-10-inference-optimization/)
- **实践**：本机用 Python 写 toy autoregressive decoder，加 KV cache 前后对比计算量；手算 7B 模型的 KV cache 显存占用
- **完成标准**：能解释 prefill 和 decode 两个阶段为什么特性不同；能手算显存占用

### 节点 08 · Batching 与调度
- **核心问题**：static batching 为什么浪费 GPU？continuous batching 如何解决？
- **参考材料**：[Orca 论文](https://www.usenix.org/conference/osdi22/presentation/yu)、[vLLM 论文](https://arxiv.org/abs/2309.06180) Section 3
- **实践**：阅读 vLLM `vllm/core/scheduler.py`，能解释 `_schedule()` 主循环逻辑
- **完成标准**：能画出 continuous batching 的请求进出图；能解释 iteration-level scheduling

### 节点 09 · PagedAttention 与显存管理
- **核心问题**：KV cache 显存碎片问题是什么？PagedAttention 如何类比 OS 虚拟内存解决？
- **参考材料**：[vLLM 论文](https://arxiv.org/abs/2309.06180)、[vLLM block_manager 源码](https://github.com/vllm-project/vllm/blob/main/vllm/core/block_manager.py)
- **环境**：AutoDL（运行 vLLM 观察显存分配）
- **实践**：阅读 block_manager 源码，解释 physical/logical block 映射；在 AutoDL 上运行 vLLM 观察显存
- **完成标准**：能解释 block 分配策略；能解释 copy-on-write 在 beam search 中的作用

### 节点 10 · FlashAttention
- **核心问题**：标准 attention 的 IO 瓶颈在哪？FlashAttention 的 tiling 如何解决？
- **参考材料**：[FlashAttention 论文](https://arxiv.org/abs/2205.14135)、[FlashAttention 源码](https://github.com/Dao-AILab/flash-attention)
- **环境**：AutoDL RTX 3090（安装 flash-attention）
- **实践**：对比标准 attention 和 FlashAttention 的显存占用与速度；读懂论文 Algorithm 1 伪代码
- **完成标准**：能手画 FlashAttention tiling 示意图；能解释 online softmax trick

### 节点 11 · 量化
- **核心问题**：INT8/FP8 量化如何减少显存？PTQ 和 QAT 的区别和代价？
- **参考材料**：[LLM.int8() 论文](https://arxiv.org/abs/2208.07339)、[GPTQ 论文](https://arxiv.org/abs/2210.17323)
- **环境**：AutoDL（用 bitsandbytes 加载量化模型）
- **实践**：在 AutoDL 上用 bitsandbytes 加载 INT8 量化模型，对比 FP16 和 INT8 的显存与速度
- **完成标准**：能解释 absmax 量化计算；能解释为什么大模型量化难（activation outlier）

### 节点 12 · 推理系统设计全景
- **核心问题**：给定延迟/吞吐/成本约束，如何选配置？各优化手段如何组合？
- **参考材料**：[vLLM 文档](https://docs.vllm.ai/)、[TensorRT-LLM 文档](https://nvidia.github.io/TensorRT-LLM/)
- **环境**：AutoDL（部署 vLLM + benchmark）
- **实践**：在 AutoDL 上用 vLLM 部署 Qwen-7B，用 benchmark 脚本测 throughput/latency，调整 batch size 观察变化
- **完成标准**：能解释 TTFT/TPOT/throughput 三个指标；能分析为什么加大 batch size 提升吞吐但增加延迟

---

## 可迁移资产

| 已有经验 | 在哪个节点用得上 |
|---------|----------------|
| OS 底层（内存管理、进程、调度） | 节点 5、6、9：显存管理和调度直接类比 |
| Gluten 向量化执行引擎（算子适配） | 节点 6、10：理解执行引擎如何接近硬件 |
| Spark Plugin 执行引擎内部经验 | 节点 8、12：理解系统级调度和执行计划 |
| Java / Spark Infra 实习 | 节点 12：系统设计取舍的工程经验 |

---

## 知识清单对照

> 权威源：[AIInfra Module 05](https://github.com/Infrasys-AI/AIInfra)、[Lilian Weng: LLM Inference Optimization](https://lilianweng.github.io/posts/2023-01-10-inference-optimization/)、[CUDA C Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
> 对照日期：2026-03-29

| 权威源章节/概念 | 覆盖情况 | 说明 |
|----------------|----------|------|
| Transformer / Attention 结构 | 已覆盖（节点 01） | — |
| C++ 内存模型 / RAII | 已覆盖（节点 02-03） | 前置差距补入 |
| CUDA thread/warp/block 执行模型 | 已覆盖（节点 04） | 前置差距补入 |
| CUDA 内存优化（coalescing / shared memory） | 已覆盖（节点 05） | 前置差距补入 |
| GPU 性能模型 / roofline | 已覆盖（节点 06） | — |
| KV Cache 优化 | 已覆盖（节点 07、09） | — |
| 批处理策略 | 已覆盖（节点 08） | — |
| PagedAttention / 显存管理 | 已覆盖（节点 09） | — |
| FlashAttention / 算子优化 | 已覆盖（节点 10） | — |
| 量化（PTQ / QAT） | 已覆盖（节点 11） | — |
| 分布式调度 / 系统设计全景 | 已覆盖（节点 12） | — |
| 长序列推理 / 并行策略 | 进阶/超纲 | 需先掌握基础系统，Phase 2 再覆盖 |
| Speculative Decoding / MoE 推理 | 进阶/超纲 | 高级加速技术，当前阶段不覆盖 |
| 知识蒸馏 / 剪枝 / 稀疏 | 不需要 | 偏模型研究方向，非推理 infra 核心 |
