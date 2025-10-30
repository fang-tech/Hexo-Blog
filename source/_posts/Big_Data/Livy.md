---
title: Livy Introduction
tags:
  - Livy
  - Spark
  - BigData
---
# 解决的问题
Livy的出现是为了提供一个统一的, 基于REST API的Spark访问层
## 没有Livy之前
> 怎么提交spark任务

- 必须登陆到服务器上, 使用spark-submit提交批处理任务
- 每次都需要重新提交任务, 如果要提交的任务的需要修改, 还需要重新上传文件

> 交互式使用-Shell模式

- 同样面临着需要进入到spark服务器才能提交任务的问题
- 同时每个用户都要开一个自己的shell, 浪费了资源
- 成员之间不能共享会话
- 不能保存配置, 以及自动化(比如覆盖配置等)

> 传统Jupyter + Spark集成方案

- 资源占用大, 每个Notebook用户都启动独立的Spark Application
- 无法同一管理Spark会话
- 无法在Notebook之间共享一个Spark会话

...

> 总体来说, 可以将问题分成以下几个方面

- 开发体验差: 无法和Jupyter等工具集成, 代码等需要频繁上传, 上下文不保存
- 不适合远程工作: 网络断开会话丢失, 无法从本地IDE连接集群
- 不适合协作工作: 团队成员之间不共享会话
- 权限控制: 必须要有服务器权限才能提交任务
- 资源管理: Livy能管理每个Spark会话使用的资源
## 有Livy之后
- 通过Jupyter Notebook向livy端发送HTTP请求建立连接并获取到sessionId
- 通过sessionId锁定到创建的sparkSession, 发送post请求携带上要执行的代码, 执行代码
- 可以在本地编写程序提交任务, 并且不需要上服务器, 便于权限管理
- 并且用户因为使用的都是自己的session, 会话之间是隔离的

# 模块介绍和流程解析
> 这里只说明core, server, rsc三个模块的内容

这三者之间的关系是
- server -> REST API 向外提供服务
- 外界 -> servlet -> livy server
- server  -> session management -> 创建InteractiveSession/BatchSession
- 这两个session类 -> RSCClient -> 发送RPC (Netty) 请求到Spark Driver中的RSCDriver
- RSCDriver -> ReplDrive/SparkContext Manager管理SparkContext

其中core模块向这两个模块提供基础公共方法和工具
## core
- scala
	- session
		- SessionState: 里面有session的各种状态
			- [[livy-session_state.canvas|livy-session_state]]

## server 

## rsc: Remote Spark Context


