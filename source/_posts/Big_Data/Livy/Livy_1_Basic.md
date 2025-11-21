---
title: Livy基础知识
tags:
  - Livy
  - BigData
  - Spark
date: 2025-10-30
---
# 概念理解

## Livy 解决了什么问题？为什么不直接用 spark-submit？
解决了用户提交spark任务的易用性和控制性问题, 使用户能通过HTTP请求向spark提交任务
- 权限问题: 使用spark-submit, 用户需要登陆上服务器, 使用spark-submit命令提交, 这将spark所在的服务器的访问权限和提交spark任务的权限绑定, 但是实际上提交spark任务的role和访问服务器的role应该是分离的.
- 易用性问题: 在过去用户需要在本地写好代码, 通过SFTP上传到服务器上, 再通过spark-submit提交任务, 现在用户可以通过jupyter + livy的方式, 在本地编写代码, 动态变更并且随时通过HTTP向Livy提交任务
- 自动化问题: 直接通过spark-submit提交任务, 无法对于spark任务进行集中管理, 捕获一些指标, 或者进行配置覆盖等其他的自动化的操作, Livy的引入相当于为用户和spark-submit之间注入了一个管理的中间层

##  Livy 支持哪几种 Session 类型？它们的区别是什么？
### Session Type
分成**Interactive Session**和**Batch Session**两类

- Interactive Session
	- 交互式Session
	- REST API端点 /sessions
	- 用户通过REST API提交**代码片段**并交互式执行
	- 支持实时获取执行结果
	- 会话通过心跳机制检测会话是否存活
- Batch Session
	- REST API端点 /batches
	- 用于提交一个**完整的spark应用程序** (需要提供 jar/py)
	- 一次性执行, 执行完后会话结束
	- 支持指定 main class 和命令行参数
	- 适合长时间运行的批处理作业
### Session Kind
分成**Spark**, **PySpark**, **SparkR**, **Shared**, **SQL**五种

- Spark/Scala: Scala语言的交互式对话
- PySpark/Python: Python语言的交互式Spark会话
- Spark/R: R语言的交互式Spark会话
- SQL: 交互式SQL Spark会话
- Shared: 共享会话 (特殊类型)

每个**交互式对话**可以同时支持所有四种语言解释器
- 创建会话的时候可以不指定kind, 而是在提交语句时指定
- 如果创建式指定了kind, 则会作为默认的语言
- 可以在同一个会话中混用不同的语言代码

## 什么是 Interactive Session？
Interactive Session是Livy提供的一种**长期运行的**, **有状态的**Spark会话, 允许用户通过REST API交互式地提供和执行代码

### 执行代码
```scala
  // InteractiveSession.scala
  def executeStatement(content: ExecuteRequest): Statement = {
    ensureRunning()
    recordActivity()

    val id = client.get.submitReplCode(content.code, content.kind.orNull).get
    val statement = client.get.getReplJobResults(id, 1).get().statements(0)
    // Send statement event to listeners.
    triggerStatementEvent(statement)
    statement
  }
```

 Interactive Session通过RSCClient (Remote Spark Context Client) 与远程的Spark Driver建立RPC连接, 提交代码到REPL解释器执行
 
 ```java
 // RSCClient.java
   public Future<Integer> submitReplCode(String code, String codeType) throws Exception {
    return deferredCall(new BaseProtocol.ReplJobRequest(code, codeType), Integer.class);
  }
  
// 向Driver提交任务
private <T> io.netty.util.concurrent.Future<T> deferredCall(final Object msg,  
    final Class<T> retType) {  
  if (driverRpc.isSuccess()) {  
    try {  
      return driverRpc.get().call(msg, retType);  
    } catch (Exception ie) {  
      throw Utils.propagate(ie);  
    }  
  }   
  // ...
  return promise;  
}
 ```

### 会话状态管理
```scala
// InteractiveSession.scala
  override def state: SessionState = {
    if (serverSideState == SessionState.Running) {
      // If session is in running state, return the repl state from RSCClient.
      client
        .flatMap(s => Option(s.getReplState))
        .map(SessionState(_))
        .getOrElse(SessionState.Busy) // If repl state is unknown, assume repl is busy.
    } else serverSideState
  }
```

- 如果状态是Running, 这个时候返回的状态 == RSCClient的状态, 否则返回serverSideState, 
- 如果状态是Running, 但是RSCClient没有给出有效返回, 这个时候假设RSCClient的状态是Busy

### 心跳机制
```scala
// InteractiveSession.scala
  override protected val heartbeatTimeout: FiniteDuration = {
    val heartbeatTimeoutInSecond = heartbeatTimeoutS
    Duration(heartbeatTimeoutInSecond, TimeUnit.SECONDS)
  }
```

心跳检测自动清理了超时的会画

### 语句管理
```scala
// InteractiveSession.scala

// 返回所有的statement
  def statements: IndexedSeq[Statement] = {
    ensureRunning()
    val r = client.get.getReplJobResults().get(
      livyConf.getTimeAsMs(LivyConf.REQUEST_TIMEOUT), TimeUnit.MILLISECONDS)
    r.statements.toIndexedSeq
  }

// 获取指定ID的的statement
  def getStatement(stmtId: Int): Option[Statement] = {
    ensureRunning()
    val r = client.get.getReplJobResults(stmtId, 1).get(
      livyConf.getTimeAsMs(LivyConf.REQUEST_TIMEOUT), TimeUnit.MILLISECONDS)
    if (r.statements.length < 1) {
      None
    } else {
      val statement = r.statements(0)
      // Send statement event to listeners.
      triggerStatementEvent(statement)
      Option(statement)
    }
  }
```

每个提交的语句片段都会被一个statement对象对应, 能从RSCClient中获取这个Statement, 读取到这个语句的信息

### 整体的流程
整体的访问流程是
Client -> Livy Server -> RSC Driver + REPL -> SparkContext
## Livy 的核心组件有哪些？

# 基本使用

1. 如何通过 REST API 创建一个 Spark Session？

- 🔍 探索：docs/rest-api.md，尝试 POST /sessions

1. 如何提交一段 Scala/Python 代码执行？

- 🔍 探索：POST /sessions/{id}/statements

1. Session 有哪些状态？如何查看当前状态？

- 🔍 探索：GET /sessions/{id}，观察 state 字段
