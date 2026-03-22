---
title: 04 · Producer：幂等写入、事务、Exactly-Once
type: note
topic: kafka
created: 2026-03-22
updated: 2026-03-23
status: completed
tags: [kafka, producer, idempotent, transaction, exactly-once, LSO]
---

# 04 · Producer：幂等写入、事务、Exactly-Once

> **目标**：理解 Kafka Producer 如何从 at-least-once 走向 exactly-once，以及幂等和事务的实现机制

---

## 核心概念

### 一、问题来源：at-least-once 的重复写入

Kafka 默认是 at-least-once：消息不会丢，但可能重复。重复的来源是网络不确定性：

```
Producer 发送消息
→ Broker 写入成功
→ 网络故障，Producer 没收到确认
→ Producer 以为失败，重试
→ Broker 收到重复消息，再次写入
```

Producer 无法区分"broker 没收到"和"broker 收到了但确认没回来"，即使 acks=all 也存在这个问题。

---

### 二、幂等 Producer：解决单 Partition 的重复写入

开启方式：`enable.idempotence=true`（Kafka 3.0 之后默认开启）

**实现机制：PID + Sequence Number**

- 每个 Producer 启动时被分配唯一的 Producer ID（PID）
- 每条消息发送时携带单调递增的序列号（Sequence Number）
- Broker 收到消息时检查 PID + Sequence Number：
  - 第一次见到这个序列号 → 正常写入
  - 已经写入过 → 丢弃，但返回成功给 Producer

无论 Producer 重试多少次，Broker 只写入一次。

**两个限制：**
1. 只保证**单个 partition** 内的幂等性，跨 partition 不在保证范围内
2. PID 是 session 级别的，Producer 重启后获得新 PID，之前的序列号记录失效，幂等性被破坏

---

### 三、事务：跨 Partition 的原子写入

事务解决的是跨 partition 的原子性——一批消息要么全部对 Consumer 可见，要么全部不可见。

**核心角色：Transaction Coordinator（TC）**

TC 是 broker 上的一个组件，负责协调事务状态。

**流程：**

```
Producer 开启事务
→ 向 TC 注册 transactional.id
→ 写入多个 partition 的消息（带事务标记）
→ 所有写入完成，向 TC 提交事务
→ TC 指示各 partition 追加 COMMIT 控制消息（append 到 log 末尾）
→ Broker 收到 COMMIT 后将该事务从未完成列表移除
→ LSO 推进
→ read_committed 的 Consumer 才能看到这批消息
```

如果中途失败，Producer abort 事务，各 partition 追加 ABORT 控制消息，Consumer 跳过这批消息。

注意：COMMIT 和 ABORT 都是**追加到 log 末尾的控制消息**，不是修改原有消息，符合日志只追加的原则。

---

### 四、LSO（Last Stable Offset）

Broker 维护 LSO = 当前所有未完成事务中，最早那条消息的 offset。

`read_committed` 的 Consumer 只能读到 LSO 之前的消息。LSO 之前保证全部是已提交或已 abort 的消息，Consumer 不需要做额外判断。

**关键风险：一个长时间未提交的事务会阻塞所有 `read_committed` Consumer。** 事务必须尽量短，不能长时间挂起。

---

### 五、Exactly-Once：幂等 + 事务 + read_committed

```
enable.idempotence=true
+ transactional.id=<唯一ID>
+ isolation.level=read_committed（Consumer 侧）
```

如果 Consumer 不配置 `read_committed`，默认是 `read_uncommitted`，会读到未提交事务中的消息。事务失败重试后，Consumer 会读到冗余的消息。

---

### 六、三种语义对比

| 语义 | 配置 | 可能丢消息 | 可能重复 | 适用场景 |
|------|------|-----------|---------|---------|
| at-most-once | acks=0 | 是 | 否 | 日志、监控等可丢场景 |
| at-least-once | acks=1 或 all，无幂等 | 否 | 是 | 大多数业务场景，消费端自己做幂等 |
| exactly-once | 幂等 + 事务 + read_committed | 否 | 否 | 金融、账务等不能容忍重复的场景 |

**实践建议：** 大多数场景用 at-least-once，在消费端做幂等处理（如数据库唯一键约束）往往比开启 Kafka 事务更简单、开销更小。事务会因为长时间未提交阻塞 Consumer，使用时需谨慎。

---

## 参考材料

1. **Kafka 设计文档 — Exactly Once**：https://kafka.apache.org/42/design/design/（找 `Exactly Once Semantics` 章节）
