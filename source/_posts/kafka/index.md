---
title: Kafka 学习主干道
type: index
topic: kafka
created: 2026-03-22
updated: 2026-03-23
status: completed
tags: [kafka, distributed-systems, messaging]
---

# Kafka 学习主干道

## 核心抽象

Kafka 是一个**持久化的分布式日志系统**，所有设计围绕一个核心约束展开：

> **消息是不可变的、有序的、可重放的日志，消费者通过 offset 自主控制读取位置**

从这个约束出发，可以推导出它的所有设计决策：

| 抽象                    | 一句话                               |
| --------------------- | --------------------------------- |
| **Topic / Partition** | 日志的逻辑分组与物理分片，分片是并发和扩展的基本单位        |
| **Offset**            | 消费者在分区日志中的位置指针，由消费者自己管理           |
| **Consumer Group**    | 一组消费者共同消费一个 topic，每个分区只被组内一个消费者消费 |
| **Replication / ISR** | 分区副本机制，ISR 决定谁有资格成为 leader        |
| **Producer Acks**     | 生产者对"写入成功"的定义，决定可靠性与延迟的取舍         |
| **Log Segment**       | 分区在磁盘上的物理存储单元，append-only，支持顺序写   |

---

## 主干道节点

| #   | 文件                                                         | 内容                                                   | 状态    |
| --- | ---------------------------------------------------------- | ---------------------------------------------------- | ----- |
| 1   | [01-core-concepts.md](../notes/01-core-concepts.md)           | Topic / Partition / Offset / Consumer Group / Broker | ✅ 已完成 |
| 2   | [02-log-and-pull.md](../notes/02-log-and-pull.md)             | 日志抽象、只追加、全局定序、Pull 模式、长轮询                            | ✅ 已完成 |
| 3   | [03-storage.md](../notes/03-storage.md)                       | Segment 结构、Page Cache、零拷贝、Retention、Log Compaction   | ✅ 已完成 |
| 4   | [04-producer-internals.md](../notes/04-producer-internals.md) | key 路由、批量发送、压缩（已合并入 06）                              | ✅ 已合并 |
| 5   | [05-replication.md](../notes/05-replication.md)               | Replica、ISR、Leader Election、acks、min.insync.replicas | ✅ 已完成 |
| 6   | [06-producer.md](../notes/06-producer.md)                     | 路由、批量、压缩、幂等、事务、LSO、Exactly-Once                      | ✅ 已完成 |
| 7   | [07-consumer.md](../notes/07-consumer.md)                     | GC、GL、Rebalance、Static Membership、Offset 提交策略        | ✅ 已完成 |
| 8   | [08-kraft.md](../notes/08-kraft.md)                           | 为什么去掉 ZooKeeper，KRaft 元数据管理                          | ✅ 已完成 |
| 9   | [09-engineering.md](../notes/09-engineering.md)               | Kafka 在真实系统中的定位与工程取舍                                 | ✅ 已完成 |

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
