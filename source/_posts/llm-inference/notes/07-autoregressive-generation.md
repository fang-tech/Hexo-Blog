---
title: 07 · 自回归生成与 KV Cache
type: note
topic: llm-inference
created: 2026-04-01
updated: 2026-04-01
status: completed
tags: [llm-inference, autoregressive, kv-cache, prefill, decode, transformer]
---

# 07 · 自回归生成与 KV Cache

> **目标**：理解 LLM 推理的完整时序过程，掌握 KV Cache 的作用和显存代价，理解 Prefill 和 Decode 两个阶段的本质差异

---

## 一、Transformer 完整计算图（回顾）

```
token 序列
    ↓ Embedding 层
[seq_len, d_model]
    ↓ Transformer Block × N 层（每层结构相同）
        ├── Multi-Head Self-Attention
        │     x → Q,K,V（各乘权重矩阵）
        │     → reshape 成 [num_heads, seq_len, head_dim]
        │     → 每个 head 独立做 Attention
        │     → 拼接 + W_O 投影
        │     → 残差连接 + LayerNorm
        └── FFN
              → W₁ 升维（×4）→ ReLU → W₂ 降维
              → 残差连接 + LayerNorm
[seq_len, d_model]
    ↓ 输出层（W_out，线性变换）
[seq_len, vocab_size]     ← 每个位置预测下一个 token 的概率分布
    ↓ 取最后一个位置（位置 seq_len-1）
[vocab_size]              ← argmax 或采样
下一个 token
```

**为什么取最后一个位置：**

位置 i 的输出预测位置 i+1 的 token。序列末尾的位置已经看过所有历史 token，它的输出就是"给定整个输入，下一个 token 是什么"。其他位置的输出在推理时直接丢弃。

---

## 二、自回归生成

**自回归（Autoregressive）**：用自己过去的输出作为下一步的输入。每一步生成的 token 被拼回输入序列，成为下一步的条件。

```
输入                    → 输出（新 token）
"今天天气"              → "真"
"今天天气真"            → "好"
"今天天气真好"          → "啊"
"今天天气真好啊"        → <EOS>（停止）
```

**为什么必须串行，不能并行输出：**

模型学到的是"给定序列，预测下一个 token 的概率分布"。第 i 个 token 的输出会改变第 i+1 个 token 的概率分布——不知道第 i 个 token 是什么，就无法确定第 i+1 个 token。这是严格的串行依赖。

---

## 三、KV Cache

**问题：** 生成第 100 个 token 时，Attention 需要计算这个 token 的 Q 和前 99 个历史 token 的 K/V 做点积。这 99 个 K/V 在前面的步骤里已经算过了，每次都重新算是浪费。

**解决：** 把每个 token 经过 W_K、W_V 变换后的向量缓存起来，下一步直接读，不重新计算。

```
第 1 步：计算 K_今、V_今 → 存入 cache
第 2 步：计算 K_天、V_天 → 追加到 cache
         直接读 cache 里的 K_今、V_今
第 3 步：计算 K_真、V_真 → 追加到 cache
         直接读 cache 里的 K_今...K_好
...
```

每步只需计算**当前新 token** 的 K/V，历史 token 的 K/V 直接从 cache 读。

**代价：** 显存随序列长度持续增长。

---

## 四、KV Cache 显存占用

```
KV Cache = 2 × seq_len × num_layers × num_heads × head_dim × bytes_per_element
           ↑
       K 和 V 各一份
```

以 Qwen-7B 为例（FP16，seq_len=2048）：

```
num_layers = 32
num_heads  = 32
head_dim   = 128
bytes      = 2（FP16）

每层 KV = 2 × 2048 × 32 × 128 × 2 = 32 MB
全部 KV = 32 MB × 32 层 = 1 GB / 请求
```

序列长度与并发数的关系（4090D 24GB，50% 显存给 KV Cache）：

```
seq_len=512   → 0.25 GB/请求 → 最多 48 个并发
seq_len=2048  → 1.00 GB/请求 → 最多 12 个并发
seq_len=4096  → 2.00 GB/请求 → 最多  6 个并发
seq_len=8192  → 4.00 GB/请求 → 最多  3 个并发
```

序列长度翻倍，KV Cache 翻倍，最大并发数减半。这是推理系统显存管理的核心约束。

---

## 五、Prefill 和 Decode 两个阶段

**Prefill（预填充）：**

输入 prompt 一次性全部处理，计算所有 token 的 K/V 填满 cache。

- 输入：N 个 token（整个 prompt）
- 并行度：高，所有 token 同时计算
- 算术强度：高（计算量 O(N²)，数据量 O(d_model²)，N 大时 compute-bound）
- 延迟指标：**TTFT（Time To First Token）**

**Decode（解码）：**

每步只生成一个新 token，读取全部历史 KV cache，追加新 K/V。

- 输入：1 个 token
- 并行度：极低，严格串行
- 算术强度：低（计算量 O(seq_len)，数据量 O(seq_len)，常数比值，memory-bound）
- 瓶颈：每步都要从显存读取整个 KV cache，GPU 算力大量闲置等数据
- 延迟指标：**TPOT（Time Per Output Token）**

| | Prefill | Decode |
|--|---------|--------|
| 输入规模 | N 个 token | 1 个 token |
| 并行度 | 高 | 极低 |
| 瓶颈 | compute-bound | memory-bound |
| 延迟指标 | TTFT | TPOT |

---

## 六、KV Cache 是后续优化的核心约束

```
节点 08（Batching）   → 多个请求共享 GPU，提升 Decode 阶段的吞吐
节点 09（PagedAttention）→ 管理 KV Cache 的显存碎片，提升最大并发数
节点 10（FlashAttention）→ 计算 Attention 时减少显存占用
```

---

## 七、跨请求 KV Cache（扩展）

节点 07 讲的是单请求内的 KV Cache（避免重复计算历史 token）。

更高层的优化是**跨请求 KV Cache**（如 LMCache）：不同请求如果有相同前缀（同一个 system prompt、同一段 RAG 背景文档），把这部分 K/V 缓存起来跨请求复用，下一个请求直接跳过这部分 Prefill 计算。两者是不同层次的概念，建立在单请求 KV Cache 之上。

---

## 参考材料

1. **vLLM 博客：Efficient Memory Management**：https://blog.vllm.ai/2023/06/20/vllm.html
2. **Lilian Weng: LLM Inference Optimization**：https://lilianweng.github.io/posts/2023-01-10-inference-optimization/
