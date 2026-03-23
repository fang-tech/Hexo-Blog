---
title: 03 · 存储设计：Segment、Page Cache、零拷贝、Retention、Log Compaction
type: note
topic: kafka
created: 2026-03-23
updated: 2026-03-23
status: completed
tags: [kafka, storage, segment, page-cache, zero-copy, retention, log-compaction]
---

# 03 · 存储设计：Segment、Page Cache、零拷贝、Retention、Log Compaction

> **目标**：理解 Kafka partition 在磁盘上的物理结构，以及高性能背后的存储设计

---

## 核心概念

### 一、Segment：Partition 的物理存储单元

一个 partition 目录下有多组文件，每组对应一个 Segment：

```
/data/kafka/order-events-0/
├── 00000000000000000000.log        ← segment 1 的消息数据
├── 00000000000000000000.index      ← segment 1 的 offset 稀疏索引
├── 00000000000000000000.timeindex  ← segment 1 的时间索引
├── 00000000000001048576.log        ← segment 2
├── 00000000000001048576.index
├── 00000000000001048576.timeindex
└── ...
```

文件名是该 segment **第一条消息的 offset**，光看文件名就知道每个 segment 的起始位置。

**三种文件的作用：**

- **`.log`**：真正存消息，append-only，消息按顺序写入
- **`.index`**：稀疏 offset 索引，不是每条消息都有记录，而是每隔若干字节记一条，用于快速定位消息的物理位置
- **`.timeindex`**：按时间戳索引，用于按时间查找消息

**Segment 切割的触发条件（满足任意一个）：**

- 文件大小达到 `log.segment.bytes`（默认 1GB）
- 最新一条消息的时间超过 `log.segment.ms`（默认 7 天）
- `.index` 文件满了（`log.index.size.max.bytes`，默认 10MB）

**为什么切割成多个 segment：**

1. **删除高效**：Retention 到期时直接删整个 segment 文件，O(1) 操作，不需要在大文件里截断
2. **查找高效**：根据 offset 快速定位 segment，再用索引缩小范围，最后顺序扫描

**完整的 offset 查找流程：**

```
目标 offset
→ 根据 segment 文件名（起始 offset）二分定位到对应 segment
→ 在该 segment 的 .index 文件里二分找到最近的稀疏索引点
→ 从索引点对应的物理位置开始顺序扫描 .log 文件
→ 找到目标 offset 的消息
```

---

### 二、Page Cache：为什么不在 JVM 堆内缓存

Kafka 刻意不在 JVM 堆内维护消息缓存，而是直接写入文件系统，依赖 OS 的 page cache。

**三个好处：**

1. **避免 GC 压力**：JVM 对象内存开销通常是原始数据的 2 倍，大堆导致 GC 停顿严重。依赖 page cache 几乎不带来额外开销
2. **OS 缓存管理更智能**：OS 自动做预读（read-ahead）和延迟写（write-behind），不需要 Kafka 自己实现
3. **进程重启不丢缓存**：broker 重启时 JVM 堆缓存全部丢失，但 OS page cache 还在，重启后无需重新预热

**实际效果**：大多数情况下 Consumer 消费的消息还在 page cache 里，拉取时根本不需要读磁盘，延迟极低。

---

### 三、零拷贝（sendfile）

**传统文件发送：四次拷贝**

```
磁盘 → 内核 page cache（DMA）→ 用户空间缓冲区（CPU）→ Socket 缓冲区（CPU）→ 网卡（DMA）
```

两次 CPU 拷贝 + 两次用户态↔内核态上下文切换。

**Kafka 使用 sendfile（Java: `FileChannel.transferTo`）：两次拷贝**

```
磁盘 → 内核 page cache（DMA）→ 网卡（DMA）
```

数据完全在内核空间流转，**CPU 拷贝从 2 次变成 0 次**。"零拷贝"的"零"指的是零次 CPU 拷贝，不是零次 DMA 拷贝。

**为什么 Kafka 能用零拷贝：**

前提是 broker 不需要修改消息内容，只是透传。所有消息处理（压缩、加密等）都交给 Producer 和 Consumer 两端，broker 保持"哑管道"。如果 broker 需要解密或转换消息，数据必须进入用户空间，零拷贝就无法使用。

---

### 四、Retention：按时间/大小整段删除

消息不因消费而删除，按 Retention 策略保留：

- **按时间**（`log.retention.hours`，默认 168 小时）：segment 内最新消息超过保留时间，整个 segment 被删除
- **按大小**（`log.retention.bytes`）：partition 总大小超过阈值，从最旧的 segment 开始删除

删除单位是整个 segment 文件，高效且简单。

---

### 五、Log Compaction：按 key 保留最新值

对每个 key，只保留最新的那条消息，旧值被后台线程清理。

```
写入：key=user1,value=A → key=user1,value=B → key=user1,value=C
Compaction 后：key=user1,value=C
```

**删除操作**：写入 value=null 的消息（tombstone），Compaction 后该 key 从 log 里消失。

**Retention vs Log Compaction：**

| | Retention | Log Compaction |
|--|-----------|----------------|
| 删除依据 | 时间 / 大小 | key 的旧值被新值覆盖 |
| 保留内容 | 一段时间内的所有消息 | 每个 key 的最新值 |
| 适用场景 | 事件流（订单、点击、日志） | 状态流（配置、用户信息） |
| 配置 | `log.retention.hours` | `log.cleanup.policy=compact` |

---

## 参考材料

1. **Kafka 设计文档 — Persistence**：https://kafka.apache.org/42/design/design/（找 `Persistence` 章节）
2. **Kafka 设计文档 — Log Compaction**：https://kafka.apache.org/42/design/design/（找 `Log Compaction` 章节）
3. **Kafka 实现文档 — Log**：https://kafka.apache.org/42/implementation/log/
4. **Kafka 实现文档 — Network Layer**：https://kafka.apache.org/42/implementation/network-layer/
