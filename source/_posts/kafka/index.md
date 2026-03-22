---
title: Kafka 学习主干道
type: index
topic: kafka
created: 2026-03-22
updated: 2026-03-23
status: in-progress
tags: [kafka, distributed-systems, messaging]
---

# Kafka 学习主干道

## 核心抽象

Kafka 是一个**持久化的分布式日志系统**，所有设计围绕一个核心约束展开：

> **消息是不可变的、有序的、可重放的日志，消费者通过 offset 自主控制读取位置**

从这个约束出发，可以推导出它的所有设计决策：

| 抽象 | 一句话 |
|------|--------|
| **Topic / Partition** | 日志的逻辑分组与物理分片，分片是并发和扩展的基本单位 |
| **Offset** | 消费者在分区日志中的位置指针，由消费者自己管理 |
| **Consumer Group** | 一组消费者共同消费一个 topic，每个分区只被组内一个消费者消费 |
| **Replication / ISR** | 分区副本机制，ISR 决定谁有资格成为 leader |
| **Producer Acks** | 生产者对"写入成功"的定义，决定可靠性与延迟的取舍 |
| **Log Segment** | 分区在磁盘上的物理存储单元，append-only，支持顺序写 |

---

## 主干道节点

| # | 文件 | 一句话 | 状态 |
|---|------|--------|------|
| 1 | [01-core-concepts.md](01-core-concepts.md) | Topic / Partition / Offset / Consumer Group 的关系与设计动机 | ✅ 已完成 |
| 2 | [02-log-and-pull.md](02-log-and-pull.md) | 日志抽象（只追加、全局定序）与 Pull 模式的设计动机 | ✅ 已完成 |
| 3 | [03-replication.md](03-replication.md) | Replica、ISR、Leader Election、acks 与数据可靠性取舍 | ✅ 已完成 |
| 4 | [04-producer.md](04-producer.md) | 幂等写入、事务、LSO、Exactly-Once 语义 | ✅ 已完成 |
| 5 | [05-consumer.md](05-consumer.md) | Consumer Group、Rebalance、Offset 管理、消费语义 | ⬜ 未开始 |
| 6 | [06-kraft.md](06-kraft.md) | KRaft 模式：为什么去掉 ZooKeeper，元数据如何管理 | ⬜ 未开始 |
| 7 | [07-engineering.md](07-engineering.md) | Kafka 在数据管道、事件驱动、训练数据流中的定位与取舍 | ⬜ 未开始 |

---

## 节点依赖

```
01-core-concepts
    ├── 02-storage-design
    ├── 03-replication
    ├── 04-producer
    └── 05-consumer
              └── 06-kraft (可并行)
                      └── 07-engineering
```
