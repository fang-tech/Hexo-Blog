---
title: 05 · Replication / ISR / Leader Election
type: note
topic: kafka
created: 2026-03-22
updated: 2026-03-23
status: completed
tags: [kafka, replication, ISR, leader-election, acks]
---

# 03 · Replication / ISR / Leader Election

> **目标**：理解 Kafka 如何通过副本机制保证数据不丢，以及 ISR、Leader Election、acks 的设计取舍

---

## 核心概念

### 一、为什么需要副本

单台 broker 随时可能挂掉。如果 partition 只存在一台 broker 上，broker 挂了数据就永远丢了。

解决方案是副本（Replica）：每个 partition 在多台 broker 上各存一份。副本数量由 `replication factor` 控制，比如 replication factor = 3，意味着这个 partition 在 3 台不同的 broker 上各有一份完整数据。

---

### 二、Leader 和 Follower

多个副本里，有且只有一个是 **Leader**，其余是 **Follower**。

- **Leader**：负责所有读写请求。Producer 写给 Leader，Consumer 也从 Leader 读。
- **Follower**：唯一职责是从 Leader 同步数据，保持冗余备份，不直接服务读写请求。

**为什么不让 Follower 分担读请求？**

从 Follower 读不会读到"错误的数据"（日志只追加，已同步的数据和 Leader 完全一致），但会读到**滞后的视图**——Follower 可能还没同步到最新的几条消息。要解决这个问题需要引入复杂的一致性协议。Kafka 的取舍是用"只读 Leader"换实现的简洁性。

（Kafka 2.4 之后引入了有条件的 Follower 读，是进阶内容。）

---

### 三、ISR

**ISR（In-Sync Replicas）** 是与 Leader 保持同步的副本集合，是一个动态维护的列表。

Follower 持续从 Leader 拉取数据。如果某个 Follower 落后超过阈值（`replica.lag.time.max.ms`），就会被踢出 ISR。追上之后重新加入。

注意：ISR 只保证"没有落后超过阈值"，不保证和 Leader 完全一致。新 Leader 选出后，可能有少量还未同步的消息丢失。

**ISR 的核心作用：只有 ISR 里的副本才有资格成为新的 Leader。**

---

### 四、Leader Election

当 Leader 所在的 broker 挂掉，Kafka 从 ISR 里选一个 Follower 升级为新 Leader，Producer 和 Consumer 自动切换。

**如果 ISR 为空怎么办？** 由 `unclean.leader.election.enable` 控制：

- **false（默认）**：只允许从 ISR 里选 Leader。ISR 为空时 partition 直接不可用——Producer 写入异常，Consumer 消费失败，直到原 Leader 的 broker 恢复上线。宁可不可用，也不丢数据。
- **true**：允许从 ISR 之外的落后副本选 Leader。Partition 继续服务，但可能有大量数据丢失。保证了可用性，牺牲了数据完整性。

这是**可用性 vs 数据完整性**的经典取舍，Kafka 默认优先保数据。

---

### 五、acks：Producer 对"写入成功"的定义

| acks | 含义 | 风险 |
|------|------|------|
| 0 | 发出去就认为成功，不等任何确认 | 可能丢消息 |
| 1 | Leader 写入成功即返回 | Leader 挂了但 Follower 未同步，消息丢失 |
| all（-1） | ISR 里所有副本写入成功才返回 | 延迟高，ISR 缩小时可能阻塞写入 |

- **acks=0 / 1**：最大化性能，接受数据丢失风险
- **acks=all**：最大化数据持久性，接受更高延迟和潜在的可用性下降

注意：acks=all 不等于绝对不丢。如果 ISR 里只剩一个副本，acks=all 退化成 acks=1。真正强持久性需要同时配置 `min.insync.replicas >= 2`，保证 ISR 里至少有 2 个副本才允许写入。

---

## 参考材料

1. **Kafka 设计文档 — Replication**：https://kafka.apache.org/42/design/design/（找 `Replication` 章节）
