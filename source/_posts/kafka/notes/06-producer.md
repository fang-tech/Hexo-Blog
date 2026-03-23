---
title: 06 · Producer：路由、批量、压缩、幂等、事务、Exactly-Once
type: note
topic: kafka
created: 2026-03-22
updated: 2026-03-23
status: completed
tags: [kafka, producer, routing, batching, compression, idempotent, transaction, exactly-once, LSO]
---

# 06 · Producer：路由、批量、压缩、幂等、事务、Exactly-Once

> **目标**：理解 Producer 的完整工作机制——从消息如何路由、如何提升吞吐，到如何保证数据可靠性

---

## 一、key 路由：消息写入哪个 Partition

Producer 直接把消息发给目标 partition 的 leader broker，没有中间路由层。路由方式有三种：

**1. 指定 key（语义分区）**
对 key 做哈希映射到某个 partition。相同 key 的消息永远进同一个 partition，实现**局部保序**。

局部保序的核心价值：**让有因果关系的事件在消费侧天然有序，简化业务逻辑。**

例：用 order_id 作为 key，同一个订单的"创建、支付、发货"三个事件全在同一个 partition，消费者按顺序读就能拿到完整的状态变更序列，不需要跨 partition 聚合。

**2. 随机分配（无 key）**
消息随机分配到各个 partition，实现负载均衡，不保证顺序。

**3. 自定义分区函数**
完全覆盖默认的 hash 策略，实现业务定制的路由逻辑。

---

## 二、批量发送：用延迟换吞吐

Producer 不是每条消息单独发一次网络请求，而是在内存里积累一批消息，满足条件后一次性发出。

两个触发条件（满足任意一个就发送）：

- **`batch.size`**（默认 16KB）：积累的消息达到这个大小
- **`linger.ms`**（默认 0ms）：等待时间超过这个值

调大 `linger.ms` 会引入少量延迟，但让更多消息积累进同一个 batch，吞吐量更高。**核心取舍：用少量额外延迟换更高吞吐。**

---

## 三、端到端批量压缩

压缩率取决于数据中的重复内容。批量压缩比单条压缩效率高得多，消息之间的相似内容（字段名、重复 key 等）可以被充分利用。

**压缩数据的生命周期：**

```
Producer 批量压缩
→ 发给 Broker（压缩状态）
→ Broker 解压验证（只做校验，不存明文）
→ 以压缩形式写入磁盘
→ 以压缩形式发给 Consumer
→ Consumer 解压
```

数据在磁盘和网络传输中全程保持压缩状态。支持算法：GZIP、Snappy、LZ4、ZStandard。

**与零拷贝的关系：互补，不冲突。** 压缩在 Producer 端完成，Broker 收到的已经是压缩好的数据，Broker 不修改内容，Broker → Consumer 这段依然可以用零拷贝。

```
Producer（压缩）→ Broker（零拷贝透传）→ Consumer（解压）
```

---

## 四、at-least-once 的重复写入问题

Kafka 默认是 at-least-once：消息不会丢，但可能重复。重复来源是网络不确定性：

```
Producer 发送消息 → Broker 写入成功
→ 网络故障，Producer 没收到确认
→ Producer 以为失败，重试
→ Broker 收到重复消息，再次写入
```

即使 acks=all 也存在这个问题——Producer 无法区分"broker 没收到"和"broker 收到了但确认没回来"。

---

## 五、幂等 Producer：解决单 Partition 的重复写入

开启方式：`enable.idempotence=true`（Kafka 3.0 之后默认开启）

**实现机制：PID + Sequence Number**

- 每个 Producer 启动时被分配唯一的 Producer ID（PID）
- 每条消息发送时携带单调递增的序列号
- Broker 收到消息时检查 PID + Sequence Number：第一次见到正常写入，已写入过则丢弃但返回成功

**两个限制：**
1. 只保证**单个 partition** 内的幂等性
2. PID 是 session 级别的，Producer 重启后幂等性失效

---

## 六、事务：跨 Partition 的原子写入

事务解决跨 partition 的原子性——一批消息要么全部对 Consumer 可见，要么全部不可见。

**Transaction Coordinator（TC）** 是 broker 上的组件，负责协调事务状态。

```
Producer 开启事务 → 向 TC 注册 transactional.id
→ 写入多个 partition 的消息（带事务标记）
→ 向 TC 提交事务
→ TC 指示各 partition 追加 COMMIT 控制消息
→ Broker 将该事务从未完成列表移除 → LSO 推进
→ read_committed 的 Consumer 才能看到这批消息
```

失败时 abort，各 partition 追加 ABORT 控制消息，Consumer 跳过。COMMIT/ABORT 都是追加到 log 末尾的控制消息，不修改原有消息。

---

## 七、LSO（Last Stable Offset）

Broker 维护 LSO = 所有未完成事务中最早那条消息的 offset。

`read_committed` 的 Consumer 只能读到 LSO 之前的消息。**关键风险：长时间未提交的事务会阻塞所有 `read_committed` Consumer。** 事务必须尽量短。

---

## 八、Exactly-Once

```
enable.idempotence=true
+ transactional.id=<唯一ID>
+ isolation.level=read_committed（Consumer 侧）
```

---

## 九、三种语义对比

| 语义 | 配置 | 可能丢消息 | 可能重复 | 适用场景 |
|------|------|-----------|---------|---------|
| at-most-once | acks=0 | 是 | 否 | 日志、监控等可丢场景 |
| at-least-once | acks=1 或 all，无幂等 | 否 | 是 | 大多数业务场景，消费端自己做幂等 |
| exactly-once | 幂等 + 事务 + read_committed | 否 | 否 | 金融、账务等不能容忍重复的场景 |

大多数场景用 at-least-once，在消费端做幂等处理（如数据库唯一键约束）往往比开启 Kafka 事务更简单、开销更小。

---

## 参考材料

1. **Kafka 设计文档 — The Producer**：https://kafka.apache.org/42/design/design/
2. **Kafka 设计文档 — Efficiency**：https://kafka.apache.org/42/design/design/
3. **Kafka 设计文档 — Exactly Once Semantics**：https://kafka.apache.org/42/design/design/
