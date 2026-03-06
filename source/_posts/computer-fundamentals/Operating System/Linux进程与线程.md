---
title: 操作系统进程管理-Linux进程与线程
categories: [Computer Fundamentals, OS, Process]
tag: [Computer Fundamentals, OS, Process]
date: 2025-08-12 22:26:26 
---

# 进程与线程

## 进程的创建

fork函数是linux中创建进程的核心函数, fork的原意是叉子, 也就是分叉. fork调用是程序执行的一个分叉点, 从这里开始, 原本的一个执行流变成了两个独立的执行流

创建的子进程继承了父进程的资源

- 打开的文件描述符
- 文件系统信息
- ...

创建的子进程在创建的时候是和父进程一样的内存空间, 会将父进程的地址空间也就是页表复制, 并复制所有的VMA, 但是标记为只读, 在修改的时候会触发page fault, 分配新的物理页, 复制数据, 更新页表项为可写

**值得一提的是这里的继承和复制都是深拷贝, 也就是会将fs_struct, mm_struct, file_struct等资源都是深拷贝, 这里虽然是继承过去了, 但是实际上已经和父进程的资源是隔离的了, 只是在最开始的时候数据是完全相同的**

### dofork

fork函数是以一个系统调用的形式存在的, 这个系统调用执行的内容就是执行dofork

```c
SYSCALL_DEFINE0(fork)
{
	return do_fork(SIGCHLD, 0, 0, NULL, NULL);
}
```

> 6.1版本及以后调用的是kernel_clone

do_fork函数传入的参数中flag是核心参数, 也是在同样是调用do_fork函数, 为什么创建进程和创建线程的时候do_fork的行为不一样, 原因就是传入的flag不一样

可传入的flag有很多

```c
// file:include/uapi/linux/sched.h
#define CLONE_VM 0x00000100 /* set if VM shared between processes */
#define CLONE_FS 0x00000200 /* set if fs info shared between processes */
#define CLONE_FILES 0x00000400 /* set if open files shared between processes */
...
#define CLONE_NEWNS 0x00020000 /* New mount namespace group */
...
#define CLONE_NEWCGROUP 0x02000000 /* New cgroup namespace */
#define CLONE_NEWUTS 0x04000000 /* New utsname namespace */
#define CLONE_NEWIPC 0x08000000 /* New ipc namespace */
#define CLONE_NEWUSER 0x10000000 /* New user namespace */
#define CLONE_NEWPID 0x20000000 /* New pid namespace */
#define CLONE_NEWNET 0x40000000 /* New network namespace */
```

- CLONE_VM: task之间共享虚拟地址空间
- CLONE_FS: task之间共享文件系统信息
- CLONE_FILES: task之间共享打开的文件描述符

还有几个是和命名空间, cgroup相关的

- CLONE_NEWS: 新任务会创建一个新的挂载点命名空间 (隔离文件系统挂载点)
- CLONE_NEWGROUP: 新任务会创建新的CGroup
- CLONE_NEWIPC: 新任务会创建新的IPC命名空间 (隔离主机名和域名)
- CLONE_NEWUTS: 创建新的UTS命名空间(隔离主机名和域名)
- CLONE_NEWUSER: 新任务创建新的User命名空间 (隔离用户ID和组ID)
- CLONE_NEWPID: 新任务创建新的PID命名空间 (隔离进程的PID)
- CLONE_NEWNET: 新任务创建新的网络命名空间 (隔离网卡设备路由表等)

> 这里创建了新的命名空间, 虽然说是"隔离", 实际上只是在可见性上做了屏蔽处理, 并不是实际的像进程的地址空间一样的完全的隔离

这里传入的SIGCHLD的含义是子进程终止后发送SIGCHLD信号通知父进程, 没有设置其他的flag

无论是do_fork函数还是6.1版本的kernel_clone, 其核心都是一个copy_process函数, 这个函数拷贝父进程的方式创建一个新的进程. 然后调用wake_up_new_task将新的进程添加到调度队列中等待调度

> copy_process

这个函数比较长, 我们分阶段说明

### 1. 复制父进程的task_struct结构体

在这一步会将父进程的task_struct完全地一模一样的, 只是复制值的, 类似浅拷贝地复制过去, 核心函数是调用dup_task_struct

申请task_struct对象的时候, 是调用的`alloc_task_struct_node(node)`, 这个函数就是调用的slab分配器从slab内核内存管理区中申请一块内存出来

这一步值得注意的是在这个时间节点, 两个task_struct是完全一样的,  mm, fs等指针都是一样的, 只是拷贝了task_struct本身, 仍然和current(父进程)指向相同的对象

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250812230544.png)

### 2. 拷贝files_struct

调用copy_files函数, 这一步会传入flag以clone_flag的形式

```c
static int copy_files(unsigned long clone_flags, struct task_struct *tsk) {
    struct files_struct *oldf, *newf;
    oldf = current->files;
    if (clone_flags & CLONE_FILES) {
        atomic_inc(&oldf->count);
        goto out;
    }
    newf = dup_fd(oldf, &error);
    tsk->files = newf;
    ...
}
```

`clone_flags & CLONE_FILES`操作, 用于检测flag中有没有CLONE_FILES的flag, 

如果有, 说明进程之间共享打开的文件描述符, 让新的进程的files_struct指向父进程的files_struct, 增加一下引用计数以后, 就通过out返回了

如果flag中没有CLONE_FILES, 说明要重新创建一个新的struct files_struct, 这个时候就会执行到dup_fd创建一个新的原本的fd的副本

1. 为新的files_struct申请内存, 调用的是kmem_cache_alloc
2. 然后对新的files_struct进行初始化, 这个新的创建的files_struct和原本的fd的值是一样

执行完毕以后, 新的进程就有自己的fd了

### 3. 拷贝fs_struct

调用copy_fs函数, 这里的逻辑和上面的copy_files是一样的

1. 检测传入的flag里面有没有CLONE_FS, 有则`fs_>user++; return`, 没有则执行copy_fs_struct创建一个父进程的副本

### 4. 拷贝mm_struct

调用copy_mm函数

1. 检测传入的flag里面有没有CLONE_VM, 如果没有会通过dup_mm申请一个新的地址空间出来, 通过allocate_mm申请了新的mm_struct, 并且将当前进程的地址空间拷贝到了新的mm_struct中用于初始化

虽然这里申请了新的地址空间, 但初始化的时候, 新的地址空间和当前进程的地址空间是完全一样, 所以子进程也能直接使用父进程中加载的可执行程序, 全局数据等(但是对于子进程来说, 这些公用的地址空间是只读的, 如果想要修改共享的地址空间, 会触发page_fault, 修改页表, 映射到新的物理内存地址上)

### 5. 拷贝进程的命名空间nsproxy

创建进程或线程的时候, 可以让内核帮我们创建独立的命名空间, 在fork系统调用中, 创建进程没有指定命名空间相关的flag, 所以新旧进程仍然是共用的一套命名空间

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250812232659.png)

### 6. 申请pid

通过alloc_pid为当前任务申请pid

在申请pid内核对象的时候, 需要传入pid_namespace, 然后创建pid_namespace->level个pid, 因为这个进程需要在每个命名空间都创建一个pid, 比如我们容器中的一个进程, 这个进程会在容器中有一个pid, 在宿主机也有一个pid, 也就是有两个pid

通过idr_alloc调用分配一个空闲的pid编号, 在3.1版本中申请进程号的函数不是idr_alloc而是alloc_bitmap.

在那个版本中, 所有的pid分配情况都是通过bitmap来管理的, bitmap最大的优点就是节省内存, 局部性很好, 但是也带来了每要获取一个没有空闲的pid都需要遍历bitmap的缺陷, 也就是获取pid是一个O(n)的操作

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250813000601.png)

这里的第三个bit是1, 也就是3这个pid被使用过了.

随着容器技术和硬件的发展, 核数和进程数快速增长, 并且内存越来越大, bitmap节省内存方面的收益已经弥补不了它获取PID O(n)的弊端, 在之后通过基数树来组织pid

> 基数树 (内核中有4bit和6bit两种, 默认使用6bit)

基数树是前缀树的一个变种, 如果使用前缀树, 我们要记录一个pid(32bit的整数)是不是使用过, 需要32层的树(实际上取决于最深的叶子节点的层高, 最大为32层). 为了降低层高, 每层树不只记录1bit的信息, 而是6bit

> 前缀树: 记录1和3使用过

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250813001439.png)

> 基数树

基数树每6bit作为一层, 也就是每层有64个槽位

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250813001913.png)

```c
struct xa_node {
	...
	unsigned char shift;
    void __rcu *slots[XA_CHUNK_SIZE];
    union {
        unsigned long tags[XA_MAX_MARKS][XA_MARK_LONGS];
    	...
    }
}
```

- shift: 表示自己在数字中的表示第几段数字, 每6bit是一段. 最低一层的shift=0, 倒数第二层 shift=6, 以此类推
- slots: 一个指针数组, 存储的是其指向的子节点的指针, 没有下一级的节点的时候指向null
- tags: 记录每个slot数组中每一个下表的存储状态, 用来表示每一个slot是否已经分配出去的状态. 一个long类型的数组, 一个long类型刚好是64bit

在基数树的基础上判断一个整数值是否存在, 或者是从这个树上分配一个新的未使用过的整数ID出来的时候, 只需要对树节点进行遍历, 分别查看每一层中的tag状态位, 看slots对应的下标是否已经占用.

### 7. 进入就绪队列

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250813003203.png)

在copy_process执行完毕的时候, 表示新进程的一个task_struct就创建出来了, 接下来内核会调用wake_up_new_task将这个新创建出来的子进程添加到就绪队列中等待调度

## 线程的创建

这里不讨论线程持有的资源有内核态和用户态共同创建之类的问题([这个问题可以看到这篇文章](./Linux虚拟内存管理#线程栈是怎么使用内存的)), 主要从内核态的角度来看操作系统是怎么创建代表线程的task_struct的, 和创建进程的task_struct有什么不同

我们这里讨论pthread(nptl)也就是glibc实现的线程库.

通过pthread_create创建线程, 最后调用到create_thread中调用系统调用do_clone

```c
static int create_thread (struct pthread *pd, ...)
{
    int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGNAL
                       | CLONE_SETTLS | CLONE_PARENT_SETTID
                       | CLONE_CHILD_CLEARTID | CLONE_SYSVSEM
                       | 0);
    int res = do_clone (pd, attr, clone_flags, start_thread,
                        STACK_VARIABLES_ARGS, 1);
    ...
}
```

这里最重要的是传入do_clone中的clone_flag, 这个flag有

- CLONE_VM: 共享虚拟地址
- CLONE_FS: 共享文件系统
- CLONE_FILES: 共享打开的文件描述符

do_clone最终会调用一段汇编程序, 进入到clone系统调用

最终clone又会调用到我们do_fork或者是kernel_clone函数中, 不过这次我们传入的flag会导致current进程和新的进程共享虚拟地址, 共享文件系统, 共享打开的文件描述符, 这里的共享是完全的共享, 也就是直接复用原进程的信息

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250812230544.png)

## 进程和线程的异同

实际上最后创建的都是一个task_struct, 并且都是通过kernel_clone函数创建task_struct, 核心区别不过是clone()传入的flag中有CLONE_VM | CLONE_FS | CLONE_FILES, 会导致子线程之间会共享成文件系统信息, 虚拟地址空间, 打开的文件描述符, 命名空间

**进程和线程之间的相同点远大于差异点**

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250813004749.png)
