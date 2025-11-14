1. 创建一个 Interactive Session 的完整流程是什么？经历了哪些步骤？

- 🔍 探索：InteractiveSession.create() → RSCClient 构建 → ContextLauncher 启动 Driver

1. PingJob 是什么？为什么需要它？

- 🔍 探索：InteractiveSession.start() 第 537 行，理解连接验证

1. Session 从 Starting 到 Running 状态，中间发生了什么？

- 🔍 探索：跟踪 PingJob 的 onJobSucceeded 回调

1. 如果 Spark Driver 启动失败，Session 会经历哪些状态？

- 🔍 探索：PingJob 的 onJobFailed 和 errorOut() 方法

#### 代码执行流程

1. 用户提交的代码是如何到达 Spark Driver 执行的？

- 🔍 探索：executeStatement() → RSCClient.submitReplCode() → RPC → ReplDriver.handle()

1. RPC 通信是如何实现的？使用了什么序列化方式？

- 🔍 探索：rsc/rpc/Rpc.java，了解 Netty + Kryo

1. Statement 的执行结果是如何返回的？

- 🔍 探索：ReplJobResults，理解同步/异步返回机制

1. Idle 和 Busy 状态是如何切换的？谁负责更新？

- 🔍 探索：RSCClient.getReplState()，Driver 端广播状态