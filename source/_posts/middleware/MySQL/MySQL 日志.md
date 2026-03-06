---
title: MySQL 日志
categories: [Middleware, MySQL]
tags: [Middleware, MySQL]
---

# MySQL日志

> 参考文章:
>   [小林coding-MySQL 日志](https://xiaolincoding.com/mysql/log/how_update.html)

执行一条update语句, 期间会发生什么

```sql
UPDATE t_user SET name = 'xiaolin' WHERE id = 1;
```

首先是前面和查询语句相似的流程

- 客户端先通过连接器建立连接, 连接器判断用户的身份
- 查询缓存, 但是因为这是一条update语句, 所以不会走查询缓存的步骤, 相反会将对应的表的缓存给清空
- 通过解析器分析update语句, 拿到update关键字, 表名等信息, 构建出来语法树, 做语法检查
- 通过预处理器判断表和字段是否存在
- 优化器确定执行计划, 这里因为是通过id作为where的条件, 会通过id这个主键执行查询
- 将执行计划交给存储引擎执行, 找到这一行, 然后执行更新

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/202507201727417.png)

而日志就在最后的更新步骤出现了

- undo log: Innodb引擎层生成的日志, 实现了事务的**原子性**, 主要实现了**事务回滚和MVCC**
- redo log: Innodb引擎层生成的日志, 实现了事务的**持久性**, 主要实现了**crush-safe**
- binlog: Server层生成的日志, 主要用于**数据备份和主从复制**

## undo log是怎么工作的

在执行DML语句的时候, 会将回滚时需要的数据都记录到undo log中

- ## insert语句, 将这个记录的主键值记录下来, 之后要回滚的时候, 就只需要将这个主键值的记录给删除就行了
- delete语句, 将这个被删除的记录的内容都记录下来, 之后回滚的时候, 就再重新插入
- update语句, 将更新列的旧值记录下来, 之后回滚的时候, 将记录重新用旧值覆盖
  - 如果不是主键列, 会记录旧值, 在回滚的时候用旧值覆盖
  - 如果是主键列, 会记录成删除旧行再插入新行, 回滚的时候也会将新记录先删除再插入原来的记录

每条记录有trx_id事务id和roll_pointer指针
- trx_id: 该记录是哪个事务修改的
- roll_pointer: 该指针将一条记录的undo log串联起来, 形成一条版本链

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/202507201752309.png)

在事务执行失败的时候, 就回滚到这个事务执行之前的版本

上面都是通过undo log实现的事务回滚, undo log同样用于MVCC

对于 \[读提交\] 和 \[可重复读\] 两个执行级别来说, 它们的快照读是通过 Read View + undo log来实现的, 区别在于创建undo log的时机不一样

- 对于读提交: 在每个select后都会生成一个Read View, 在这个隔离级别中, 不会有脏读现象, 但是如果其他事务修改了数据并且提交了, 该事务中就能读到, 也就会出现同一个select语句前后执行的结果不一样的问题
- 对于可重复度: 在事务启动的时候生成一个Read View, 保证了在事务中, 不会读到其他事务修改的数据

## 为什么需要redo log

redo log是用于解决崩溃恢复问题, 也就是数据的持久化(这里包括undo log和表数据)

redo log是物理日志, 会记录对某个数据页进行了什么修改, 比如对**XXX表空间中的YYY数据页ZZZ偏移量的地方做了AAA更新**

> 为什么要写到redo log, 而不直接将数据写到磁盘中, 多此一举

redo log是使用了WAL(Write-Ahaed Logging)技术, MySQL的写操作并不是立刻写到磁盘上, 而是先写日志, 然后写到磁盘上. 

如果直接将数据写入到磁盘中, 就是**随机写**, 而将数据写入到log文件中因为是追加写, 所以是**顺序写**, 磁盘的I/O性能有10~100倍的提升

> 产生的redo log是直接写入到磁盘中吗

问出来很明显就不是了, 我们需要注意到内核的缓冲区这个存在, 我们执行write函数, 实际上是将数据写入到内核缓冲区中, 直到我们主动调用fsync()或者内核自己同步, 才会刷盘. 这是其中一方面, 另一方面是redo log本身就有一个redo log buffer, 用于缓冲log, 提高IO性能

### redo log什么时候刷盘

redo log刷盘的时机主要有下面几个(注意是刷盘, 而不是写入到文件中, 前者是直接写入到磁盘中的, 后者只是将内容写入到内核缓冲区中)
- MySQL正常关闭的时候
- redo log buffer中的记录写入量大于redo log buffer内存空间的一半的时候
- 每次事务提交的时候都将缓存在redo log buffer中的redo log直接持久化到磁盘中(这个策略通过innodb_flush_log_at_trx_commit参数控制)

> innodb_flush_log_at_trx_commit 参数控制的是什么

- 参数为0的时候: 每次事务提交会将redo log留在redo log buffer
- 参数为1的时候: 每次事务提交会将redo log buffer中的redo log直接持久化到磁盘中
- 参数为2的时候: 每次事务提交会将redo log buffer中的redo log写入到redo log文件中(并不是持久化到了磁盘中, 而是到了内核中的Page Cache)

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/202507211006233.png)

> 那么innodb_flush_log_at_trx_commit参数的值是0或者2的时候, 什么时候刷盘?

InnoDB后台线程每隔一秒:

- 参数为0的时候: 后台线程执行write()将redo log buffer中的redo log写到Page Cache中, 再执行fsync()写入到磁盘中, 所以**MySQL进程的崩溃, 会导致前1s的所有事务丢失**
- 参数为2的时候: 区别就是只需要调用fsync(), 数据已经都在Page Cache中了, 所以**MySQL进程的崩溃, 不会导致事务丢失, 只有宕机崩溃的时候, 会导致前1s的所有事务丢失**

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/202507211013253.png)

### redo文件写满了怎么办

InnoDB中redo log是以重做文件日志组的形式存在的, redo log group中有两个redo log文件, 两个文件大小一致, 分别叫做 `ib_logfile0`和`ib_logfile1`

重做文件日志组是采用循环写的模式, 从头开始写, 写到了末尾又回到了开头, InnoDB引擎会先写 `ib_logfile0`, `然后写ib_logfile1`, 当 `ib_logfile1`也写满的时候, 就会重新回到logfile0的开头

InnoDB使用write pos 表示redo log当前记录写到的位置, checkpoint 表示当前要擦除的位置, 其中要擦除的位置就是已经持久化到磁盘中脏页的位置

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/202507211025398.png)

如果write pos 追上了 checkpoint, 说明redo log已经满了, 这个时候MySQL会被阻塞, 会停下来将Buffer Pool中的脏页刷新到磁盘中, 将redo log中对应的记录标记并清除, 然后移动checkpoint, MySQL恢复正常

### redo log的格式

redo log是物理日志, 记录的是对数据页的修改, redo log的格式是:

```
LSN | Log Record Type | Log Record Length | Log Record Body
```
- LSN: Log Sequence Number, redo log的序列号, 用于标识redo log的顺序
- Log Record Type: redo log的类型, 比如更新数据页, 删除数据页
- Log Record Length: redo log的长度, 用于标识redo log的大小
- Log Record Body: redo log的具体内容, 包含了对数据页的修改信息
Log Record Body的内容是根据Log Record Type的不同而不同的, 比如更新数据页的Log Record Body会包含更新前的数据和更新后的数据, 删除数据页的Log Record Body会包含被删除的数据

## 为什么需要binlog

binlog是逻辑日志, 记录的是对数据的修改, binlog的主要作用是用于数据备份和主从复制

MySQL在执行完一条更新操作以后, Server层还会生成一条binlog, 等事务提交的时候, 会将该事务执行过程中产生的binlog写入到binlog文件中

binlog只能用于归档, 是不具备crush-safe能力的

### redo log和binlog的区别

1. 适用对象不同

- binlog是Server层实现的日志, 所有的存储引擎都能使用
- redo log是InnoDB引擎实现的日志, 只能用于InnoDB存储引擎

2. 文件格式不同

- binlog有 3 种格式: STATEMENT, ROW, MIXED
  - STATEMENT: 记录每一条更新的操作的SQL语句, 这也是为什么binlog会被称作逻辑日志, 主从复制种slave端就是通过binlog中记录的SQL语句来执行更新操作的. 但是会存在动态函数的问题, 比如NOW()函数, 在主从复制中, 主库执行的NOW()函数和从库执行的NOW()函数的时间不一致, 会导致数据不一致
  - ROW: 记录行数据最后被修改成什么样子(这种格式的日志就不是逻辑日志了), 缺点就是如果是批量更新, 会记录每一行的修改, 可能会导致binlog文件过大
  - MIXED: 结合了STATEMENT和ROW两种格式, 会根据不同的情况自动选是使用STATMENT模式还是ROW模式

3. 写入方式不同

- binlog是追加写, 写满一个文件, 就创建一个新的文件继续写, 不会覆盖之前的文件, 保存的是全量的日志
- redo log是循环写, 写满一个文件, 就回到开头继续写, 只保存最近的日志

4. 用途不同

- binlog主要用于数据备份和主从复制, 也可以用于数据恢复
- redo log主要用于崩溃恢复, 也可以用于数据恢复

> 如果整个数据库的数据都被删除了, 能通过redo log来恢复吗

不能, 只能通过binlog来恢复, redo log记录的不是全量的数据, 如果脏页被写入到了磁盘中, 就会从redo log中删除

##  主从复制是怎么实现的

MySQL集群的主从复制可以分成三个阶段

- 写入binlog: 主库写binlog日志, 提交事务并更新本地存储数据
- 同步binlog: 从库通过IO线程连接到主库, 从主库获取binlog日志, 写入到本地的中继日志中
- 回访binlog: 从库通过SQL线程读取中继日志, 执行SQL语句, 更新本地存储数据

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/202507211046273.png)

详细过程是

- 主库在收到的客户端提交事务的请求之后, 会先写入到binlog中, 然后提交事务, 更新存储引擎中的数据, 事务提交完成后, 返回给客户端"操作成功"的响应
- 从库创建一个专门的IO线程, 连接主库的log dump线程, 来接收主库的binlog日志, 再把binlog信息写入到relay log中, 再返回给主库复制的复制成功的响应
- 从库会创建一个用于回放binlog的线程, 去读relay log中的中继日志, 然后回放binlog更新存储引擎中数据

> 从库是不是越多越好

并不是, 从库越多, 主库需要创建同样多的log dump线程来处理从库连接上来的IO线程, 对主库的资源消耗较高

> 主从复制还有哪些模型

- 同步模型: MySQL主库提交事务的线程需要等待所有的从库都响应复制成功以后, 才会返回给客户端结果, 这样的形式性能和可用性都很差
- 异步模型: 就是上面的模型, 缺点就是如果主库宕机, 就会出现数据的丢失
- 半同步模型: 介于上面两者之间, 事务线程不用等待所有的从库返回复制成功响应, 只需要等待一部分返回复制成功响应就会返回给客户端. 通过这种形式, 保证了即使主库宕机, 也还有从库保存有最新的数据, 不会导致数据丢失

### binlog什么时候刷盘

在Server层, binlog有一个binlog cache, 事务提交的时候, 是先写到binlog cache中, 然后再把binlog cache写入到binlog文件中.

一个线程中只能有一个事务正在执行, 每个线程也都携带了一个binlog cache

> 什么时候binlog cache会被写入到binlog中

在事务提交的时候, 会将binlog cache中的完整事务写入到binlog文件中, 并清空binlog cache (注意这里不是持久化, 也就是只调用了write(), 而没有调用fsync())

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/202507211137495.png)

MySQL中提供了sync_binlog参数来控制数据库binlog刷盘的频率

- sync_binlog = 0的时候, 表示每次提交事务都只write, 不fsync(), 什么时候fsync交给操作系统决定
- sync_binlog = N的时候, 表示每次提交事务的时候都只write(), 在积累了N个事务以后fsync

MySQL中默认的是sync_binlog = 0

> 小结update语句的执行过程

这里我们从优化器分析出来了成本最小的执行计划以后, 执行器按照其计划执行

1. 执行器调用存储引擎的接口, 通过主键索引树搜索id = 1这一行的记录
   - 如果id = 1这一行所在的数据也本身就在buffer pool中, 就直接返回给执行器更新
   - 如果不在, 就需要先将数据页从磁盘中加载到buffer pool中, 返回记录给执行器更新
2. 执行器拿到聚簇索引记录以后, 比较更新的记录和原记录是不是一样的
   - 如果是一样的, 不执行后续流程
   - 不一样, 将更新前的记录和更新后的记录都当作参数传给InnoDB层, 让InnoDB执行真正的更新操作
3. 开启事务, InnoDB更新前需要先记录相应的undo log, 因为是更新操作, 所以将旧值存入undo log, 将undo log写入buffer pool中的Undo页面, 然后将这个Undo页面记录到redo log中
4. InnoDB层开始更新记录, 先更新内存中记录, 并标记成脏页, 然后将记录写入到redo log中, 这时就是更新完成了, 后台线程将内存中脏页刷盘
5. 开始记录该语句对应的binlog, binlog先被保存到binlog cache中, 在事务提交的时候统一将该事务运行过程中的所有binlog更新到磁盘上
6. 最后就是事务提交, 也就是\[两阶段提交\], 接下来就是讲两阶段提交

## 为什么需要两阶段提交

在主从复制的场景中, 主库MySQL宕机了, redo log负责的是主库的崩溃恢复, binlog负责的是事务能否传播到从库

而redo log和binlog持久化到磁盘是两个独立逻辑, 那么就会出现下面的两种情况

- **如果在redo log刷盘以后, MySQL宕机了, binlog还没有来得及刷盘**. 这个时候, 崩溃恢复以后, 主库中能将Buffer Pool中的事务修改的数据恢复, 但是从库读binlog并没有执行之前的事务, 出现了**主库是新值, 从库是旧值**
- **如果在binlog刷盘以后, MySQl宕机了, redo log没有刷盘**. 就会变成**主库是旧值, 从库是新值**

这两种情况都是出现了主从库数据不一致的问题

MySQL给出的解决方案就是两阶段提交, 将事务提交的过程分成了两个阶段, 准备阶段和提交阶段

### 两阶段提交是怎么样的

为了保证binlog和redo log的一致性, MySQL使用了内部XA事务

在客户端执行commit的时候, MySQL内部开启一个XA事务, 分两阶段完成XA事务的提交

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/202507211200934.png)

将redo log的写入拆分成了两个阶段: prepare和commit, 中间穿插binlog

- prepare阶段: 将XID (内部XA事务的ID) 写入到redo log, 将redo log对应的事务状态设置成prepare, 将redo log持久化到磁盘(innodb_flush_log_at_trx_commit = 1的时候)
- commit阶段: 将XID写入到binlog中, 然后将binlog持久化到磁盘 (sync_binlog = 1的时候), 接着调用引擎的提交事务接口, 将redo log状态设置成 commit

### 异常重启后的现象

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/202507211205211.png)

在MySQL重启以后, 会按照顺序扫描redo log文件, 碰到处于 prepare 状态的 redo log, 就会拿着redo log的XID去binlog中查看是否有该XID

- 如果没有XID, 说明redo log完成了刷盘, 但是binlog还没有完成刷盘, 回滚事务, 因为从库会无法同步这个数据, 对于到途中的A时刻
- 如果有XID, 说明redo log和binlog都完成了刷盘, 则提交事务, 对应B时刻崩溃恢复

**对于处于prepare阶段的redo log, 既可以提交事务, 也可以回滚事务, 这取决于是否在binlog中查找到与redo log相同的XID**

**两阶的提交是以binlog写成功为事务提交成功的标志**

### 两阶段提交的问题

- 磁盘的I/O次数高, 每次事务提交都会有两次刷盘 (如果在两个刷盘配置都是1的情况下)
- 锁竞争激烈, 多事务的场景下, 需要通过锁来保证提交的原子性(即保证binlog的写入顺序和事务提交顺序一致)

### 组提交

为了解决锁竞争激烈的问题, 引入了binlog组提交机制, 当有多个事务提交的时候, 将多个binlog刷盘操作合并成一个, 从而减少磁盘I/O的次数

将commit拆分成了三个阶段

- flush阶段: 多个事务按照进入的顺序将binlog从binlog cache写入到文件(不刷盘)
- sync阶段: 将多个事务的binlog合并一次刷盘
- commit阶段: 各个事务按顺序做commit操作

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/202507211220153.png)

在每个阶段引入了锁机制以后, 锁就只针对每个队列进行保护, 不再锁住提交事务的整个过程, 可以看出来, **锁粒度减小了, 这样就使得多个阶段可以并发执行**

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/202507211225877.png)

> flush 阶段

flush阶段的队列用于支撑redo log的组提交

> sync 阶段

事务会在sync队列上等待一定时间, 以组合更多事务的binlog然后一起刷盘

sync队列的作用是用于支持binlog的提交

> commit 阶段

承接sync阶段的事务, 完成最后的引擎提交

