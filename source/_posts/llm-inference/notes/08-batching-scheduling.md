---
title: 08 · Batching 与调度
type: note
topic: llm-inference
created: 2026-04-01
updated: 2026-04-01
status: completed
tags: [llm-inference, batching, continuous-batching, scheduling, throughput, vllm]
---

# 08 · Batching 与调度

> **目标**：理解 Batching 的本质收益，掌握 Static Batching 和 Continuous Batching 的区别，理解 vLLM scheduler 的核心调度逻辑

---

## 一、为什么需要 Batching

Decode 阶段每步只处理 1 个 token，GPU 有上万个 CUDA core，但只有 1 个 token 的计算量——绝大多数算力在空转。

**Batching 的本质收益：让多个请求分摊读取模型权重矩阵的开销，提升算术强度。**

用 roofline 语言：

```
单请求 decode：
  计算量：1 × d_model²
  数据量：d_model²（读权重矩阵）
  算术强度 ≈ 1 FLOP/Byte   ← 极低，memory-bound

batch size = N：
  计算量：N × d_model²
  数据量：d_model²（权重矩阵只读一次，N 个请求共用）
  算术强度 ≈ N FLOP/Byte   ← 随 batch size 线性增长
```

batch size 越大，算术强度越高，在 roofline 图上向右移动，逐渐从 memory-bound 逼近 ridge point。

---

## 二、Static Batching 的问题

Static Batching：收集一批请求，打包给 GPU，等所有请求都完成再接受下一批。

```
Batch = [请求A（输出10个token）, 请求B（输出20个token）, 请求C（输出5个token）]

请求A完成后：████████████████░░░░░░░  （空等请求B）
请求C完成后：█████░░░░░░░░░░░░░░░░░░  （空等请求B）
请求B完成后：████████████████████████  整个 batch 才能结束
```

问题：LLM 的输出长度在生成完成前完全不可预知，短请求完成后只能空等最长的请求，GPU slot 浪费。

---

## 三、Continuous Batching

Orca 论文（OSDI 2022）提出：**不以请求为调度单位，以每个 decode step 为调度单位（iteration-level scheduling）**。

每个 decode step 结束后：
- 检查哪些请求输出了 `<EOS>`，完成的立即出队返回结果
- 把等待队列里的新请求加入 running batch

```
Step 1：[请求A, 请求B, 请求C] → 各生成 1 个 token
Step 3：请求C 输出 <EOS> → 移除，加入请求D
        [请求A, 请求B, 请求D]
Step 4：请求A 输出 <EOS> → 移除，加入请求E
        [请求B, 请求D, 请求E]
...
```

GPU 始终满载，短请求完成即返回，不被长请求拖累。

| | Static Batching | Continuous Batching |
|--|----------------|---------------------|
| 调度单位 | 整个请求 | 每个 decode step |
| 短请求完成后 | 等待 batch 里最长的请求 | 立即换入新请求 |
| GPU 利用率 | 低（有空等） | 高（持续满载） |
| 算术强度 | 随空等下降 | 始终最大化 |

---

## 四、为什么 2022 年才出现

不是没人想到，是之前没有这个需求。

传统深度学习推理（图像分类、机器翻译）的输出长度固定或可预估，Static Batching 够用。

LLM 自回归生成的输出长度完全不可预知——"帮我写首诗"可能 50 个 token，"帮我分析报告"可能 2000 个 token，差距 40 倍。这个特性在 ChatGPT 大规模商业化之前没有真实需求驱动。Orca 论文恰好在 ChatGPT 上线前几个月发表，正是 LLM 推理服务化开始被认真对待的时间点。

OS 的调度思想早就有，但需要有人把它迁移到 LLM 推理这个新场景，识别出问题根本是"输出长度不可预知"，然后用 iteration-level scheduling 解决。

---

## 五、vLLM Scheduler 的核心逻辑

vLLM 实现了 Continuous Batching，调度器每个 step 做三件事：

**1. 决定哪些请求进入 running**：受显存容量和 batch size 上限约束

**2. Swap out**：显存不足时，把低优先级请求的 KV Cache 换出到 CPU 内存，等显存释放后换回

**3. Preempt**：高优先级请求进来时，暂停低优先级请求

```
每个 decode step：
    running 队列 → 执行一步 decode
    等待队列 → 尝试加入 running（显存够才允许）
    running → 完成的请求移出，返回结果
```

---

## 六、Continuous Batching 残留的问题

Continuous Batching 解决了调度效率，但没有解决**显存碎片**问题：

传统做法给每个请求预分配连续显存，大小 = 最大序列长度 × KV cache per token：

```
请求A：预分配 2048 token 的空间（实际只用 50 token）→ 大量浪费
请求B：预分配 2048 token 的空间（实际用 1800 token）→ 勉强够用
```

- 不知道请求最终输出多少 token，不知道预留多少显存
- 预留少了不够，预留多了浪费
- 显存碎片导致实际并发数远低于理论上限

这是节点 09（PagedAttention）要解决的核心问题。

---

## 参考材料

1. **Orca 论文**：https://www.usenix.org/conference/osdi22/presentation/yu
2. **vLLM 论文 Section 3**：https://arxiv.org/abs/2309.06180
