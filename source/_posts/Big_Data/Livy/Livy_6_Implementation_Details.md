#### RPC 机制

1. RPC 的消息格式是什么？如何序列化和反序列化？

- 🔍 探索：KryoMessageCodec.java，消息头和消息体

1. RPC 如何处理超时和重试？

- 🔍 探索：Rpc.call() 方法，Future 机制

1. SASL 握手的流程是什么？

- 🔍 探索：SaslHandler.java，客户端和服务端的认证

1. 如何保证 RPC 连接的安全性？

- 🔍 探索：secret 的生成和验证

#### REPL 实现

1. Scala REPL 是如何集成的？

- 🔍 探索：ScalaInterpreter.scala，IMain 的使用

1. PySpark 是如何执行的？与 Scala 有何不同？

- 🔍 探索：PythonInterpreter.scala，py4j 的使用

1. 如何实现代码自动补全？

- 🔍 探索：completion() 方法，REPL 的 complete 接口

1. 如何取消正在执行的 Statement？

- 🔍 探索：cancelStatement()，线程中断机制

#### SparkApp 监控

1. YARN 应用状态是如何监控的？

- 🔍 探索：SparkYarnApp.yarnAppMonitorThread，轮询机制

1. 如何从日志中提取 Application ID？

- 🔍 探索：searchAppIdFromLog() 方法，正则匹配

1. 如何处理 Spark 应用的异常退出？

- 🔍 探索：isProcessErrExit() 方法