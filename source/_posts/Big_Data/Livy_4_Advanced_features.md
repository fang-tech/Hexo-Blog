#### 状态管理

1. 为什么 InteractiveSession 需要双层状态（serverSideState + replState）？

- 🔍 探索：state 属性的实现，理解细粒度状态监控

1. 状态转换有哪些保护机制？为什么需要这些保护？

- 🔍 探索：transition() 方法的 areSameStates 和 transitFromInactiveToActive 检查

1. 多个线程同时触发状态转换时，如何保证一致性？

- 🔍 探索：synchronized 关键字的使用

1. SparkApp 状态和 Session 状态的映射关系是什么？

- 🔍 探索：stateChanged() 回调，理解事件驱动的状态同步

#### 会话恢复

1. Livy Server 重启后，如何恢复已有的 Session？

- 🔍 探索：InteractiveSession.recover()，SessionStore 的实现

1. 支持哪些持久化方式？它们的优缺点是什么？

- 🔍 探索：FileSystemStateStore、ZooKeeperStateStore、JDBCStateStore

1. 恢复时如何重新连接到 Spark Driver？

- 🔍 探索：rscDriverUri 的保存和使用

1. 什么情况下 Session 无法恢复？

- 🔍 探索：InteractiveRecoveryMetadata.isRecoverable() 方法

#### 资源管理

1. Livy 如何检测 Session 超时？

- 🔍 探索：SessionHeartbeat trait，心跳机制

1. 如何动态添加 Jar 和 File？它们被分发到哪里？

- 🔍 探索：addJar()、addFile()，理解资源上传到 HDFS

1. 命名 Session (Named Session) 是什么？与普通 Session 有何不同？

- 🔍 探索：driverName 参数，Thrift Server 集成

1. 如何限制并发 Session 数量？

- 🔍 探索：SessionCounter.scala，SESSION_MAX_CREATION 配置