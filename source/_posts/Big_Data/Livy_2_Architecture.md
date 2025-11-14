#### 整体架构

1. Livy 的整体架构是怎样的？有哪几层？

- 🔍 探索：画出 REST API → Server → RSCClient → RSCDriver → Spark 的层次图

1. Livy Server 和 Spark Driver 是什么关系？

- 🔍 探索：理解 Livy Server 是客户端，Spark Driver 是服务端的反转架构

1. RSC (Remote Spark Context) 是什么？为什么需要它？

- 🔍 探索：rsc/src/main/java/org/apache/livy/rsc/README.md

1. Livy 如何与 YARN/K8s 集成？

- 🔍 探索：SparkYarnApp.scala，查看如何监控 YARN 应用

#### 模块职责

1. core、server、rsc、repl 四个模块各自的职责是什么？

- 🔍 探索：查看各模块的 pom.xml 依赖关系

1. 为什么 Interactive Session 需要 ReplDriver，而 Batch Session 不需要？

- 🔍 探索：对比 ReplDriver.scala 和 BatchSession.scala

1. Livy 如何支持多种 Spark 版本？

- 🔍 探索：LivyConf 中的 LIVY_SPARK_VERSION 配置