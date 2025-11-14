#### 配置管理

1. 有哪些关键的 Livy 配置项？如何调优？

- 🔍 探索：livy.conf.template，理解各配置项的作用

1. 如何配置多个 Spark 版本支持？

- 🔍 探索：livy.spark.versions 配置

1. 如何配置 Kerberos 认证？

- 🔍 探索：LivyServer.scala 中的 UserGroupInformation

1. 如何配置资源隔离（队列、用户模拟）？

- 🔍 探索：AccessManager、proxyUser 参数

#### 监控告警

1. Livy 暴露了哪些监控指标？如何采集？

- 🔍 探索：Metrics.scala，Prometheus 集成

1. 如何监控 Session 的健康状态？

- 🔍 探索：/sessions API，状态监控

1. 如何收集 Spark Driver 的日志？

- 🔍 探索：logLines() 方法，SparkApp.log()

1. 如何知道 Session 启动时间过长？

- 🔍 探索：MetricsKey.INTERACTIVE_SESSION_START_TIME
#### 高可用

1. Livy 的高可用方案是什么？

- 🔍 探索：ZooKeeper 集成，主备切换

1. 如何实现多 Livy Server 的负载均衡？

- 🔍 探索：ClusterManager、SessionAllocator

1. Session 恢复失败时，如何处理？

- 🔍 探索：Recovering 状态的异常处理