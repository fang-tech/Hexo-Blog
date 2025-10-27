# Hive的作用
Hive将提交任务这一件事情简化, 屏蔽了执行引擎, 提供了HiveQL供用户使用, 用户可以使用SQL的形式提交并执行任务.

从输入数据到最后的执行经过的步骤:

- 用户通过客户端连接到HiveServer2提交SQL -> Driver接受客户端的HiveQL -> Compiler / Semantic Analyzer / Optimizer将QL转化成逻辑计划, 物理计划, 并进行CBO优化 -> Execution Engine将物理计划分解成具体的执行任务, 提交到底层的计算引擎上 (MapReduce, Spark) -> 在HDFS上存储数据, 由YARN分配资源给计算引擎
- 另一条线是HiveServer2 -> Hive MetaStore Server访问元数据, 通过Thrift提供元数据 (表/分区/统计), 并将结果持久化到关系型数据库

## HMS (Hive Metastore Server)
HMS