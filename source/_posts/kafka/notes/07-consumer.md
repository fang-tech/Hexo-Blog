---
title: 07 · Consumer：Rebalance 与 Offset 提交策略
type: note
topic: kafka
created: 2026-03-23
updated: 2026-03-23
status: completed
tags: [kafka, consumer, rebalance, static-membership, offset, group-coordinator, commit]
---

# 05 · Consumer：Rebalance 与 Offset 提交策略

> **目标**：理解 Consumer Group 如何协调 partition 分配，以及 offset 提交策略的取舍

---

## 核心概念

### 一、Group Coordinator 与 Group Leader

**Group Coordinator（GC）**：某个 Broker 上的组件，负责管理 Consumer Group 的成员关系和协调流程。不负责计算分配方案，只负责协调过程和下发结果。

**Group Leader（GL）**：Coordinator 从 JoinGroup 的成员中选出的一个 Consumer，负责计算 partition 分配方案，计算完成后发回给 Coordinator，再由 Coordinator 通知所有成员。

```
Consumer 启动
→ 找到 Group Coordinator（特定 broker）
→ 发送 JoinGroup 请求
→ Coordinator 选出 Group Leader
→ Group Leader 计算 partition 分配方案
→ 发回给 Coordinator
→ Coordinator 把方案下发给所有 Consumer
→ 每个 Consumer 开始消费分配到的 partition
```

**为什么由 Group Leader 计算而不是 Coordinator？**
分配策略是可插拔的（轮询、按 key、自定义等），放在客户端计算更灵活，Coordinator 只需协调流程，不需要理解业务分配逻辑。

---

### 二、Rebalance：触发时机与代价

**触发时机：**
- 有新的 Consumer 加入 group
- 有 Consumer 正常离开 group
- Consumer 心跳超时（Coordinator 认为它已死）
- Group 订阅的 topic 新增或减少 partition

**心跳超时是最常见的坑：** Consumer 没有崩溃，只是处理一批消息太慢，长时间没发心跳，Coordinator 认为它死了，触发 Rebalance，把它的 partition 分给别人。Consumer 处理完想提交 offset，发现自己已经不属于这个 group 了，提交失败。解决方式是合理配置 `max.poll.interval.ms` 和 `session.timeout.ms`。

**经典协议的全局同步屏障（Kafka 4.0 之前）：**

Rebalance 开始时，所有 Consumer 必须停止消费，等所有成员重新发送 JoinGroup，Group Leader 重新计算分配方案，Coordinator 下发，所有成员响应，才能继续消费。

代价高的两个原因：
1. **范围太大**：任何一个成员的变化都让整个 group 全部停下来
2. **等最慢的节点**：整个过程需要等所有成员响应，group 越大，越容易遇到慢节点，整体耗时越长

**Kafka 4.0 新协议（KIP-848）：**

增量式分配，只调整受影响的 partition，未受影响的 Consumer 继续消费，不再需要全局同步屏障。配置 `group.protocol=consumer`，Kafka 4.0 默认开启。

---

### 三、Offset 提交策略

Consumer 消费后需要提交 offset 到 `__consumer_offsets`。

**1. 自动提交（enable.auto.commit=true，默认）**

每隔 `auto.commit.interval.ms`（默认 5000ms）在后台自动提交。

两个问题：
- **重复消费**：两次提交之间 Consumer 崩溃，这段时间的消息重启后重新消费（at-least-once）
- **消息丢失（更严重）**：消息被拉下来还在异步处理中，自动提交定时器触发，offset 提交成功，Consumer 崩溃。重启后从新 offset 继续，那批还没处理完的消息永远丢失——消息被提交了但没处理完

**2. 手动提交**

- **commitSync（同步提交）**：处理完一批后调用，阻塞等待提交成功再继续。安全不丢，但吞吐低。
- **commitAsync（异步提交）**：提交请求发出后不等结果，继续处理下一批。吞吐高，但失败时**不自动重试**——用旧 offset 重试可能覆盖已成功提交的新 offset，导致重复消费。

**实践中的结合方式：**
```
正常消费循环：commitAsync（保持吞吐）
Consumer 关闭时：commitSync（确保最后一次提交成功）
```

---

### 四、auto.offset.reset

新 Consumer Group 第一次消费，或 offset 已过期被清理时的行为：

| 配置值 | 行为 |
|--------|------|
| latest（默认） | 从当前最新消息开始，历史消息全部忽略 |
| earliest | 从最早消息开始，消费所有历史 |
| none | 直接抛异常，要求必须有已知 offset |

---

### 五、Static Membership：减少不必要的 Rebalance

**问题背景：**

默认情况下，Consumer 每次重启（滚动部署、JVM crash）都会触发两次 Rebalance——旧成员离开一次，新成员加入一次。对有状态的流处理应用（Kafka Streams）代价极高，不仅消费暂停，本地处理状态还要重新构建。

**解法：给每个 Consumer 实例配置持久化的 Group Instance ID**

```
ConsumerConfig.GROUP_INSTANCE_ID_CONFIG = "consumer-instance-1"
```

Coordinator 识别这个固定 ID：
- **Consumer 重启**：Coordinator 发现是同一个 Instance ID 回来了，不触发 Rebalance，直接恢复之前的 partition 分配
- **Consumer 真正离线**（超过 `session.timeout.ms`）：才会触发 Rebalance

| | 动态 Membership（默认） | Static Membership |
|--|----------------------|-------------------|
| Consumer 重启 | 触发 Rebalance | 不触发 Rebalance |
| partition 分配 | 重启后重新分配 | 重启后恢复原有分配 |
| 适用场景 | 无状态消费者 | 有状态流处理、滚动部署 |

注意：同一 group 内 Instance ID 必须唯一，重复会导致旧实例被强制踢出。需要 broker 版本 >= 2.3。

**等待期间的取舍：**

Static Membership 下，Consumer 重启回来之前，对应的 partition 暂时没有消费者，消息堆积，LAG 增加。这是有意识的取舍：

- **用单个 partition 的短暂空窗**，换**整个 group 不需要 Rebalance**
- 能接受短暂堆积但不能接受 Rebalance 代价（有状态流处理）→ 用 Static Membership
- 不能接受任何堆积（高实时性无状态消费）→ 用默认动态 Membership

等待时间就是 `session.timeout.ms`，不是额外的新配置。Coordinator 在这个窗口期内，如果同一 Instance ID 回来了，直接恢复原分配；超时才判定死亡并触发 Rebalance。

---

## 参考材料

1. **Kafka 4.0 Consumer Rebalance Protocol**：https://kafka.apache.org/42/operations/consumer-rebalance-protocol/
2. **Kafka Consumer 配置**：https://kafka.apache.org/42/configuration/consumer-configs/
3. **Kafka 设计文档 — Static Membership**：https://kafka.apache.org/42/design/design/（找 `The Consumer` → `Static Membership`）
