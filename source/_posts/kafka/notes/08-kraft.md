---
title: 08 · KRaft：为什么去掉 ZooKeeper，元数据如何管理
type: note
topic: kafka
created: 2026-03-23
updated: 2026-03-23
status: completed
tags: [kafka, kraft, zookeeper, raft, metadata, controller]
---

# 08 · KRaft：为什么去掉 ZooKeeper，元数据如何管理

> **目标**：理解 ZooKeeper 模式的根本问题，以及 KRaft 如何用内部机制替代它

---

## 一、ZooKeeper 模式是什么

Kafka 4.0 之前，集群元数据（哪些 broker 在线、partition leader 是谁、ISR 列表、topic 配置）由外部的 **ZooKeeper** 集群管理。运维 Kafka 意味着同时运维两套系统。

---

## 二、ZooKeeper 模式的三个核心问题

**1. 元数据状态经常不一致**

ZooKeeper 用 watch 机制监控元数据变化，但 watch 机制性能有限，经常导致 ZooKeeper 里的元数据和 broker 内存里的状态不同步。同时，ZooKeeper 的推送通知不是原子操作，可能有些 broker 收到了变更，有些没收到，导致不同 broker 对集群状态的认知出现分歧。

**2. 扩展性瓶颈**

所有元数据都存在 ZooKeeper 里，ZooKeeper 的容量和性能成了 Kafka 集群规模的上限，直接限制了单个集群能支持的 partition 数量。

**3. 运维复杂**

两套系统意味着两套配置、监控、安全策略和故障排查流程，资源消耗大，部署维护成本高。

---

## 三、KRaft 的解法

**不再依赖外部系统，用 Kafka 内部的 Controller 节点和 Raft 协议管理元数据。**

元数据存成一个 **metadata log**，本质上就是一个内部 topic：
- **Producer**：Active Controller（负责写入元数据变更）
- **Consumer**：所有 broker（主动 pull，订阅元数据更新）

```
ZooKeeper 模式：Kafka ←→ ZooKeeper（外部推送）
KRaft 模式：  broker（pull）← metadata log ← Controller（内置 Raft）
```

---

## 四、Controller Quorum 与 Raft

通常选 3 或 5 个节点作为 controller，用 Raft 协议选出一个 Active Controller 处理所有元数据写入，其他 controller 是 follower。

**为什么选奇数？** Raft 是多数派协议，奇数个节点总能出现多数派：
- 3 个 controller：容忍 1 个节点故障
- 5 个 controller：容忍 2 个节点故障
- 偶数节点（如 4 个）：容忍 1 个故障，但成本比 3 个高，没有额外收益

Raft 保证 metadata log 的全局写入顺序一致，所有 broker 看到的元数据变更顺序完全相同。

---

## 五、元数据一致性：KRaft vs ZooKeeper

**ZooKeeper 模式的不一致是结构性的**：推送机制本身不保证所有 broker 同时收到变更，可能出现"跳着应用"的情况。

**KRaft 的不一致是暂时性的**：broker 按顺序消费 metadata log，可能落后几条 entry，但最终会追上，且追上过程是有序的，不会出现跳过或乱序。

**原子性的边界**：单条 metadata log entry 是原子的（Raft 多数派提交保证），但跨多条 entry 的逻辑操作没有跨条目的原子性。这比 ZooKeeper 的推送更可靠，但不是完全的事务语义。

---

## 六、KRaft 带来的改进

| | ZooKeeper 模式 | KRaft 模式 |
|--|--------------|------------|
| 外部依赖 | 需要独立 ZooKeeper 集群 | 无外部依赖 |
| 元数据一致性 | 结构性不一致，可能永久分歧 | 暂时性落后，最终有序追上 |
| 扩展性 | partition 数量受 ZooKeeper 限制 | 理论上支持百万级 partition |
| 运维复杂度 | 两套系统 | 一套系统 |
| 单节点部署 | 不支持 | 支持（开发环境友好） |

Kafka 4.0 起完全移除 ZooKeeper 支持，只支持 KRaft 模式。

---

## 参考材料

1. **Kafka KRaft 运维文档**：https://kafka.apache.org/42/operations/kraft/
2. **KIP-500（设计动机）**：https://cwiki.apache.org/confluence/display/KAFKA/KIP-500
