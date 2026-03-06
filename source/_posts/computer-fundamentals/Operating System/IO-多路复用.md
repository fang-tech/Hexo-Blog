---
title: 操作系统网络IO-Linux中的同步阻塞IO与多路复用epoll
categories: [Computer Fundamentals, OS, Netwokr-IO]
tag: [Computer Fundamentals, OS, Netwokr-IO]
date: 2025-08-14 21:54:26
---

# Linux中的同步阻塞IO与多路复用epoll

## Linux中的同步阻塞IO

在完成了socket的创建, 三次握手的过程以后, 通信双方建立了连接, 接下来的过程就是通信的过程了, 相互传递数据, 使用同步阻塞进行通信的过程其实就是简单地直接使用`recv`阻塞式地等待数据发送到来, 发送到了再接收处理

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250816120305.png)

### 等待接收消息

通过`strace`能发现`recv`执行的系统调用是`recvform`, 进入到系统调用以后, 用户进程就进入到了内核态, 执行一系列的内核协议层函数, 然后到socket对象的接收队列中查看是否有数据, 没有数据的话, 就把自己添加到socket对应的等待队列中. 最后让出CPU

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250816120332.png)

这里的关键是`recvfrom`最后是怎么将自己的进程给阻塞调用的

在`recvfrom`系统调用中根据fd找到对应的socket对象以后, 调用`sock_recvmsg`

`sock_recvmsg `-> `__sock_recvmsg` -> `__sock_recvmsg_nosec`

在最后的方法中, 调用到了socket->ops->recvmsg

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250816120355.png)

由socket对象图, 我们能知道这个时候`recvmsg`指向的是`inet_recvmsg`函数

在这里我们又要调用socket对象中的sock的sk_prot中的`recvmsg`方法, 同样我们能从socket对象图中回忆起来这个方法指向是`tcp_recvmsg`

> tcp_recvmsg 执行过程

1. 遍历接收队列接收数据
2. 如果接收队列中的数据长度 >= target, 接收数据
3. 反之, 没有收到足够长度的数据, 启用`sk_wait_data`阻塞掉当前进程

接下来就是看到sk_wait_data是怎么将进程阻塞掉的

> sk_wait_data执行过程

1. 创建等待队列项wait
2. 调用DEFINE_WAIT宏, 将当前进程(current)关联到所定义的等待队列项上, 并且为这个等待队列项注册了回调函数`autoremove_wake_function`
3. 调用`sk_sleep`获取socket对象下的等待队列列表头wait_queue_head_t
4. 调用`prepare_to_wait`来把新定义的等待队列项wait插入sock对象的等待队列, 并设置进程的状态为TASK_INTERRUPTIBLE
5. 最后调用`schedule_timeout`让出CPU, 然后进行睡眠

当内核收完数据产生就绪事件的时候, 就可以查找socket等待队列上的等待项, 进而可以找到回调函数(`autoremove_wake_function`)和在等待该socket就绪事件的进程了. **这里我们产生了一次进程上下文切换的开销**

### 软中断模块

接下来让我们转换视角, 从来到负责接收和处理数据包的软中断这里来

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250816120520.png)

我们直接看到TCP协议的接收函数`tcp_v4_rcv`上

> tcp_v4_rcv

1. 获取数据帧的ip header, tcp header
2. 根据数据包header中的信息, 查找到对应的socket
3. 然后进入到tcp_v4_do_rcv主体函数

> tcp_v4_do_rcv

如果连接状态是ESTABLISHED, 进入到tcp_rcv_established函数处理

> tcp_rcv_established

1. 调用`tcp_queue_rcv`将接收数据放到接收队列中
   1. 这个函数将接收到的数据放入到socket的接收队列的队尾
2. 数据准备好, 调用`sock_def_readble`唤醒socket上阻塞掉的进程
   1. 这里实际上调用的是sk_data_ready函数指针, 但是在创建socket的时候, 在sock_init_data函数里面, 已经将这个函数指针指向了sock_def_readable函数了, 它是默认的数据就绪处理函数
   2. 这里会从socket->wait等待队列上获取到等待的wait项, 并从中获取到我们在之前的`recvfrom`中调用的`DEFINE_WAIT(wait)`的时候关联的进程
   3. 调用`wake_up_interruptible_sync_poll`来唤醒socket上因为等待数据被阻塞的进程

> wake_up_interruptible_sync_poll

这是一个宏函数, 实际上会执行到`__wake_up_sync_key`上, 这个函数最后调用`__wake_up_common`实现唤醒, 这个时候其中的nr_exclusive参数传入的是1, 这是指即使有多个进程都阻塞在同一个socket上, 也只唤醒一个进程

在`__wake_up_common`中, 会招出来一个等待队列项curr, 然后调用其curr->func, 这里实际上调用的函数是我们在`DEFINE_WAIT`的时候设置的`autoremove_wake_function`

在`autoremove_wake_function`中调用到了`default_wake_function`, 最后调用到`try_wake_up(curr->private)`, 这里的private就是我们在之前的`DEFINE_WAIT`关联的被阻塞的进程. 在这个函数执行完以后, 进程就被推入到可运行队列里面了. 这里我们又将产生了一次进程上下文切换的开销

### 小结

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250816120154.png)

这个模式基本没有在使用了, 因为性能差开销大. 一个线程也只能维护一个连接, 将维持的连接的数量和线程的数量绑定, 连接的数量少还好, 如果是10K的连接, 岂不是要创建10K个线程, 没有哪台机器能扛得住, 同时在IO密集型任务里面, 使用同步阻塞的方式会导致频繁的进程上下文的切换, 这也导致性能极差. 

但是也不是说这个方式一无是处, 编程的方式简单这个优点对比Epoll尤其明显 (不要把实现简单不当成优势, 程序员也是人, 需要在有限的时间里实现特定的功能, 在实现的难度、 性能等方面进行多方面的权衡后选择一个合适的技术实现才是正确的方式)

## 多路复用Epoll

现在我们正式进入到这个章节的重点, 多路复用, 首先我们要明确这里复用的内容是什么, 是进程(线程, task), 我们想实现在一个进程里面同时监听并处理多个连接, 这就是对于进程的复用.

epoll的存在我们可以想象成一只牧羊犬. 在没有牧羊犬之前, 人一次只能放一只羊, 等这只羊吃完草以后才能放别的🐏. 在有了牧羊犬以后, 我们将🐏交给牧羊犬管理, 一旦有那只羊有什么情况, 由牧羊犬统一通知我们, 这样我们就能实现管理多只🐏.

一个简单的例子 (伪代码)

```c
int main() {
	listen(lfd, ...);
    cfd1 = accept(...);
    cfd2 = accept(...);
    efd = epoll_create(...);
    
    // 不光可以将accept socket添加在epoll中, 监听socket同样能添加到其中
    epoll_ctl(efd, EPOLL_CTL_ADD, cfd1, ...);
    epoll_ctl(efd, EPOLL_CTL_ADD, cfd2, ...);
    epoll_Wait(efd, ...);
}
```

可以看到epoll关键的函数是

- `epoll_create`: 创建一个epoll对象
- `epoll_ctl`: 向epoll对象添加要管理的连接
- `epoll_wait`: 等待其管理的连接上的IO事件

接下来我们挨个解析这些函数

### epoll内核对象的创建

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250816140652.png)

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250816140712.png)

这个数据结构的几个成员的含义

- wq: 等待队列链表, 软中断数据就绪的时候会通过wq来找到阻塞在epoll对象上的用户进程
- rbr: 红黑树, 为了支持对海量连接的高效查找, 插入, 删除操作, epoll内部使用了红黑树, 用于管理用户进程下添加进来的所有socket进程
- rdllist: 就绪的描述符的链表, 当有连接就绪的时候, 内核就会把就绪的连接放入到这个链表中, 这样应用进程就只需要判断链表就能找到就绪连接, 不需要遍历整棵树

epoll_create中执行的主要函数就是ep_alloc从slab内存管理器中申请一个epoll出来并将其初始化, 这里的初始化工作其实就是将其中的各个成员都初始化一遍

### 为epoll添加socket

这里我们只考虑使用EPOLL_CTL_ADD添加socket, 忽略删除和更新

现在假设和客户端的多个连接的socket都创建好了, 并且也创建好了socket对象, 要使用epoll_ctl注册每一个socket的时候, 内核会做的事

- 分配一个红黑树节点对象epitem
- 将等待事件添加到socket的等待队列中, 回调函数是`ep_poll_callback`
- 将epitem插入到epoll对象的红黑树

> 看到epoll_ctl系统调用

1. 根据传入的fd找到eventpoll, socket内核对象
2. 对于EPOLL_CTL_ADD操作来说, 会执行到ep_insert函数, 所有的注册都是在这个函数中完成的

> ep_insert

1. 分配并初始化epitem
2. 设置socket等待队列
3. 将epi插入到eventpoll对象的红黑树中

#### 分配并初始化epitem

epitem也就是红黑树节点指向的数据结构

```c
struct epitem{
	// 红黑树节点
    struct rb_node rbn;
    
    // socket文件描述符信息
    struct epoll_filefd ffd;
    
    // 所归属的eventpoll对象
    struct eventpoll *ep;
    
    // 等待队列
    struct list_head pwqlist;
}
```

这里主要是将epi->ep设置传入的struct eventpoll, 然后用要添加的socket的file和fd填充到epi->ffd中.

也就是一个epitem对象在初始化以后会关联到一个eventpoll对象和一个socket(file, fd)

#### 设置socket等待队列

在完成了epitem的创建并初始化了以后, 就要设置socket对象上的等待任务队列并设置回调函数

看到ep_item_poll函数

这个函数会通过`ep_item_poll` -> `epi->ffd.file->f_op->poll(这里是struct file中的f_op, 在这里实际指向的是sock_poll函数)` -> `sock->ops->poll(这里实际指向的是tcp_poll)` -> `sock_poll_wait`

在最后的sock_poll_wait会传入由sk_sleep获取的sock对象下的等待队列列表头wait_queue_head_t, 这个也是前面同步阻塞中`sk_wait_data函数`获取等待队列列表头的方法, 这里是为插入等待队列项做准备

`sock_wait `-> `poll_wait `-> `p->qproc`(实际上调用的是在`ep_insert`中设置成的`ep_ptable_queue_proc`函数)

在`ep_ptable_queue_proc`中, 新建了一个等待队列项, 并注册其为回调函数`ep_poll_callback`函数, 然后将这个等待项添加到socket的等待队列中 (注意这里不是epoll的等待队列, 是socket的等待队列)

这个时候等待队列项是交给epoll来管理的, 所以不需要像之前的recvfrom里面一样将private设置成当前进程在socket就绪的时候唤醒进程, 这里会设置成NULL

这样设置以后, 在软中断将数据收到socket的接收队列以后, 会通过注册的ep_poll_callback函数回调, 进而通知epoll对象

#### 插入红黑树

在分配完epitem对象以后, 就将其插入到红黑树中

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250816140735.png)

为什么使用红黑树? 为了能让epoll在查找效率, 插入效率, 内存开销等多个方面比较均衡, 只考虑查找效率的话, 肯定使用哈希表了, 谁能查得过它.

### epoll_wait等待接收

当它被调用的时候, 观察eventpoll->rdllist链表中有没有数据, 有数据就返回, 没有数据就创建一个等待队列项, 将其添加到eventpoll的等待队列, 然后就将自己阻塞掉. 这里的过程比较像之前recvfrom中创建socket的等待队列项的过程

1. 会定义等待队列项 并关联current(当前进程), 这里设置的回调函数是default_wake_function
2. 添加到epoll对象的等待队列链表中
3. 将自己阻塞了, 让出CPU, 选择下一个进程调度

这样设置以后, 实际上在触发软中断以后会将epoll关联的进程给唤醒, 然后再让epoll去处理它管理的socket们, 将自己的rdllist中的数据返回

> 也就是当没有IO事件的时候, epoll也是会将自己阻塞的, 毕竟没有任务, 占着CPU干嘛, 在epoll中. 其实epoll本身是阻塞的, 只是将socket设置成了非阻塞

### 数据来了

在epoll_ctl运行完以后为sock添加了等待队列项, 回调函数是`ep_poll_callback`, 这里的private指向的是NULL, 因为socket不再关联进程

在epoll_wait运行完以后, 又为epoll添加了等待队列项, 回调函数是`default_wake_function`, 这里private指向的就是进程了.

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250816140258.png)

接下来就是看到软中断处理完以后, 怎么依次进入到各个回调函数中, 最后通知到用户进程的

#### 将数据收到任务队列中

和前面的同步阻塞中的软中断模块一样, 我们从tcp_v4_rcv进入, 最后通过tcp_queue_rcv将数据保存到socket的接收队列中. 

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250816120520.png)

#### 查找就绪回调函数

走完了左边的保存数据的路径以后, 接下来就是唤醒等待队列上的进程的路径, 同样我们最终调用到`sock_def_readable`这个默认的数据就绪处理函数

以`sock_def_readable`这个函数为入口, 从socket->wait等待队列上获取到等待的wait项, 这里会得到我们之前在`epoll_ctl`中设置的`ep_poll_callback`回调函数

> sock_def_readable

1. 通过`wq_has_sleeper`判断sock等待队列不为空
2. 不为空的话, 通过`wake_up_interruptible_sync_poll`执行等待队列项上的回调函数

> wake_up_interruptible_sync_poll

这个函数和在同步阻塞中一样, 调用到`__wake_up_common`, 并从中获取到等待队列项curr, 然后调用curr->func. 这里就是在`ep_insert`中设置的`ep_poll_callback`了

#### 执行socket就绪回调函数

> ep_poll_callback

1. 获取到wait对应的epitem
2. 获取epitem对应的eventpoll结构体
3. 将当前epitem添加到eventpoll的就绪队列上, 也就是eventpoll->rdllist
4. 查看eventpoll的等待队列上是否有等待, 如果有等待就调用`wake_up_locked`将epoll对应的进程唤醒

> wake_up_locked

这里实际上最后也是调用到wake_up_interruptible_sync_poll获取等待队列项, 然后调用curr->func, 不过这个时候调用到的是epoll的等待队列项中的回调函default_wake_function, 将epoll唤醒

> default_wake_function

这个函数会重新将epoll_wait进程推入可运行队列,  等待内核重新调度. 当这个进程重新运行的时候, 会从epoll_wait阻塞时暂停的代码处继续执行, 将rdllist中就绪的事件返回给用户进程

### 小结

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250816140551.png)
