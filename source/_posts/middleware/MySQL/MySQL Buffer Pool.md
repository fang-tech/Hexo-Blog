---
title: MySQL Buffer
categories: [Middleware, MySQL]
tags: [Middleware, MySQL]
---

# Buffer Pool

就像CPU Cache一样的缓存中间层, 来提高MySQL的性能, 写方面采用写回策略(修改的时候如果在缓存中, 就会直接修改缓存中的数据, 然后标记为脏页)

## Buffer Pool缓存了什么

在MySQL启动的时候, 会为Buffer Pool分配一片连续的内存空间, 然后按照默认的`16KB`的大小分页, Buffer Pool中的页就是缓存页

Buffer Pool中缓存了这六种信息

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250820224844.png)

> undo页记录的是什么

生成undo log的时候, undo log会写入到Buffer Pool中的Undo 页面

> 查询一条记录, 就只会缓冲一条记录吗

会将整个页都缓存进去