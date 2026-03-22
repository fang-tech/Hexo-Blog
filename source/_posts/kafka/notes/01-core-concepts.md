---
title: 01 · Core Concepts
type: note
topic: kafka
created: 2026-03-22
updated: 2026-03-23
status: completed
tags: [kafka, topic, partition, offset, consumer-group]
---

# 01 · Core Concepts

> **目标**：理解 Kafka 的四个核心抽象及其关系，能画出消息从 produce 到 consume 的完整路径

---

## 核心概念

### Topic

Topic 是逻辑概念，是消息的命名分类单位（如 `order-events`、`user-clicks`）。Topic 本身不存储数据，真正存储数据的是 partition。

### Partition

Partition 是物理存储单元。每个 partition 在磁盘上是一个**目录**，目录下包含多个 segment 文件（`.log`、`.index`、`.timeindex`）。Kafka 按大小或时间把数据切割成多个 segment，避免单个文件无限增长。

Partition 解决两个问题：

- **并行度**：多个 partition 分布在不同 broker 上，读写可以同时进行，吞吐量线性扩展。消费端同理，N 个 partition 最多支持 N 个消费者并行消费。
- **水平扩展**：partition 是 Kafka 的最小分布单元，不同 partition 可以在不同 broker 上，单个 topic 的数据量不受单台机器限制。

注意：**partition 内部消息严格有序，partition 之间没有顺序保证。** 相同 key 的消息会路由到同一个 partition，这是实现"同一实体的事件有序"的机制。

### Offset

Offset 是消费者在 partition 中的位置指针，是一个递增整数，存在于 **partition 级**（不是 topic 级）。

**Offset 由消费者自己管理，不是 broker 管。** 消费者定期把当前 offset checkpoint 到 Kafka 内部的特殊 topic `__consumer_offsets`，崩溃重启后从上次 checkpoint 的位置继续消费。

传统消息系统让 broker 追踪消费状态，带来两难困境：
- broker 立刻标记已消费 → consumer 崩溃会丢消息
- 等 consumer 确认 → 确认前崩溃会重复消费，且 broker 要为每条消息维护多个状态，性能差

Kafka 的解法让状态极简：每个 partition 只需一个整数。代价是 **at-least-once** 语义——消息不会丢，但可能重复消费（处理完但提交 offset 前崩溃）。Exactly-once 需要额外机制（幂等 producer + 事务）。

Offset 带来的副作用也是核心特性：**消费者可以主动倒回任意 offset 重放历史消息**，传统队列消费后即删除做不到这点。

### Consumer Group

Consumer Group 是一组消费者的逻辑分组，共同消费一个 topic。

**核心规则：同一个 group 内，每个 partition 同一时刻只被一个消费者消费。**

这个约束是 offset 能用单个整数表示的前提——如果多个消费者同时消费同一个 partition，就需要复杂协调机制，状态就不再是一个简单数字了。

两种消费模式：

- **组内（负载均衡）**：partition 在组内消费者间分配，每条消息只被一个消费者处理。消费者数量不能超过 partition 数量，超出的消费者会 idle（没有 partition 可分配）。
- **组间（广播）**：不同 group 独立维护各自的 offset，同一条消息对每个 group 完整可见。

判断是否同一个 group 的标准：**这些消费者是在协作处理同一件事（用同一个 group），还是独立处理不同的事（用不同的 group）？**

例：库存系统和物流系统都消费 `order-events`，它们是**不同的 group**——各自需要完整看到所有订单事件，互不影响。

---

## 消息从 produce 到 consume 的完整路径

```
Producer
  → 按 key 路由到指定 partition（相同 key → 同一 partition）
  → 写入 broker 上的 partition（append-only log）
  → Consumer 从 broker pull 数据（指定 offset）
  → 处理消息
  → 提交新的 offset 到 __consumer_offsets
```

---

## 参考材料

1. **Kafka 官方文档 — Introduction**：https://kafka.apache.org/42/getting-started/introduction
2. **Kafka 设计文档 — The Consumer**：https://kafka.apache.org/42/design/design/（找 `The Consumer → Consumer Position`）
3. **LinkedIn 工程博客 — The Log**：https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying

---

## CLI 验证

```bash
# 进入容器（apache/kafka:4.2.0）
docker exec -it <container-name> bash

# 创建 topic，3 个 partition
/opt/kafka/bin/kafka-topics.sh --create --topic test-topic \
  --partitions 3 --replication-factor 1 \
  --bootstrap-server localhost:9092

# 查看 partition 分布
/opt/kafka/bin/kafka-topics.sh --describe --topic test-topic \
  --bootstrap-server localhost:9092

# 查看 consumer group offset 状态
/opt/kafka/bin/kafka-consumer-groups.sh --describe --group group-A \
  --bootstrap-server localhost:9092
# 观察：CURRENT-OFFSET / LOG-END-OFFSET / LAG 三个字段
```
