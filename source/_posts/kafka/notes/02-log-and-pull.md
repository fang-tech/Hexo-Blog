---
title: 02 · 日志抽象与 Push vs Pull
type: note
topic: kafka
created: 2026-03-22
updated: 2026-03-23
status: completed
tags: [kafka, log, append-only, pull, long-poll]
---

# 02 · 日志抽象与 Push vs Pull

> **目标**：理解 Kafka 的存储基础——日志抽象，以及消费模型选择 Pull 的设计动机

---

## 核心概念

### 一、什么是日志

日志是最简单的存储抽象：**只追加、全局有序、按时间排列的记录序列。**

每条记录追加到末尾，每条记录被分配一个唯一的单调递增序号。这个序号定义了一种"时间"概念——序号小的记录比序号大的旧。

**关键性质：这种"时间"与物理时钟无关。**

在分布式系统里，不同机器的物理时钟不同步，即使同步了，同一毫秒内发生的两件事也无法用物理时间判断先后。日志序号解决的是**全局定序**问题——整个系统里所有事件有一个统一的、无歧义的顺序。这是 Kafka 能作为"事件的单一来源"的基础。

注意区分两种"日志"：
- **应用日志**（log4j、syslog）：给人读的非结构化文本
- **数据日志**（Kafka partition）：给程序读的结构化序列，是这里讨论的概念

---

### 二、只追加带来的三个好处

**1. 顺序写性能极高**

磁盘随机写慢，顺序写可以接近内存速度。日志只追加意味着所有写入都是顺序的，Kafka 因此能在普通机械硬盘上达到很高的吞吐量。

**2. 消息不因消费而删除，天然支持重放**

传统队列消费一条删一条。Kafka 的 partition 只追加，消息按 retention 策略保留（如 7 天）。重放不是 Kafka 特意实现的功能，而是日志本来就不删的自然结果——结合 offset 机制，回退 offset 即可重放。

**3. offset 是一个整数就够了**

只追加保证了日志是连续递增的有序序列。offset 作为分界线，能将序列精确地划分成两部分：offset 之前是已消费的，offset 之后是未消费的。

如果日志可以修改或删除过去的消息，这个分界线就会失效——offset 左边的内容可能已经变了，一个整数无法再准确表达消费进度。

---

### 三、Push vs Pull

**Push 模式的根本问题：broker 控制速率，消费者被动接收。**

生产速率超过消费速率时，消费者会被压垮。要解决这个问题，broker 需要感知每个消费者的处理能力并动态调整——类似 TCP 的流量控制，实现复杂，且不同消费者能力差异大，broker 很难精确控制。

**Pull 模式的优点：消费者自己控制节奏。**

消费者落后了慢慢追，能力强就多拉，broker 完全不需要感知消费者状态。同时天然支持批量拉取——每次拉取当前 offset 之后所有可用消息，批量大小由消费者决定。

**Pull 模式引入的新问题：空轮询。**

broker 没有数据时，消费者不断发请求，broker 不断回"没有数据"，白白消耗资源。

**Kafka 的解决方案：长轮询（long poll）。**

消费者发出请求后，如果 broker 暂时没有数据，请求阻塞等待，直到有数据或超时才返回。保留了 Pull 的优点，同时避免了空轮询的浪费。

---

## 参考材料

1. **LinkedIn 工程博客 — The Log**：https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying（Part One: What is a log?）
2. **Kafka 设计文档 — Push vs Pull**：https://kafka.apache.org/42/design/design/（找 `The Consumer → Push vs. pull`）
