---
title: 操作系统进程管理-进程实现原理
categories: [Computer Fundamentals, OS, Process]
tag: [Computer Fundamentals, OS, Process]
date: 2025-08-11 13:57:26 
---

# 进程实现原理

进程是一个程序运行时的实例, 一个程序要运行起来, 需要硬盘, 内存, CPU, 网络等资源, 如果这些部分都有用户手动来管理, 开发一个程序会变成一个极其繁琐和困难的事情, 操作系统针对这些程序运行时需要的资源抽象出来了进程这个概念. 进程持有并统一管理所有一个程序要运行时需要的资源

对于资源的集合, 在概念中被称为PCB(Process Control Block), 而在Linux中对应的内核对象就是task_struct这个数据结构

```c
struct task_struct {
    // 1. 进程的状态
    volatile long state;
    
    // 2. 进程的pid
    pid_t pid;
    pid_t tgid;
    
    //3. 和进程树的关系 (父进程, 子进程, 兄弟进程)
    struct task_struct __rcu *parent;
    struct listhead children;
    struct listhead sibling;
    struct task_struct *group_header;
    
    // 4. 进程优先级
    int prio, static_prio, normal_prio;
    unsigned int rt_priority;
    
    // 5. 进程地址空间
    struct mm_struct *mm, *active_mm;
    
    // 6. 进程文件系统信息 (当前目录...)
    struct fs_struct *fs;
    
    // 7. 进程打开的文件描述符
    struct files_struct *files;
    
    // 8. namespace
    struct nsproxy *nsproxy;
}
```

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250811143531.png)

## 进程的特性

接下来更进一步地讲解进程中的中的各个特性, 也基本是围绕进程持有的资源展开

### 进程状态

```c
// 1. 进程的状态
volatile long state;
```

通过top命令查看, 其中的S一列就是进程的State

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250811224718.png)

进程的简单的状态转化图

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250811225155.png)

其他的状态

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250811225235.png)

### PID和进程树

```c
// 2. 进程的pid
pid_t pid;
pid_t tgid;
```

这里的tgid是实际的进程的PID, 调用getgid()返回的也是tgid, pid在单线程的时候等于tgid, 在多线程的时候, 属于同一个进程的每个线程的tgid都是一样的, pid不同.

通过`ps -ef`命令能看到所有正在运行的进程

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250811225336.png)

其中的PID一栏就是该进程的PID, PPID是其父进程的PID (PID是0的进程是systemd进程)

```c
//3. 和进程树的关系 (父进程, 子进程, 兄弟进程)
struct task_struct __rcu *parent;
struct listhead children;
struct listhead sibling;
struct task_struct *group_header;
```

```bash
#pstree
```

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250811225553.png)

进程通过task_struct中的parent和children两个task_struct对象, 将进程按照父子关系组合成了一棵树, 其中所有的进程的最终父进程都是systemd (PID=0)

### 进程调度

```c
// 4. 进程优先级
int prio, static_prio, normal_prio;
unsigned int rt_priority;
```

调度器分成实时调度器和完全公平调度器, 前者常用于内核task, 后者常用于用户态的task, 运行的时间会收到prio的影响, 这一部分内容会在后面详细说明

### 进程内存地址空间

```c
// 5. 进程地址空间
struct mm_struct *mm, *active_mm;
```

```c
struct mm_struct {
    struct vm_area_struct * mmap; // 链表
    struct rb_root mm_rb;	 // 红黑树
    
    // 进程中的各个逻辑段的地址
    unsigned long mmap_base;
    unsigned long task_size;
    unsigned long start_code, end_code, start_data, end_data;
    unsigned long start_brk, brk, start_stack;
    unsigned long arg_start, arg_end, env_start, env_end;
}
```

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250811231602.png)

所有的进程共用相同的内核内存区域, 并且内核内存是通过物理内存直接映射分配的

同时对于内核线程, 它的task_struct中的mm是null, 因为它没有用户态的虚拟地址空间

### 进程文件系统

```c
// 6. 进程文件系统信息 (当前目录...)
struct fs_struct *fs;
```

```c
struct fs_struct {
    ...;
    struct path root, pwd;
}

struct path {
    struct vfdsmount *mnt;
    struct dentry *dentry;
}
```

不同于接下来要介绍的打开的文件, 这个属性记录的是pwd(当前工作目录), root(根目录)等进程在文件系统的位置

### 进程文件打开列表

```c
// 7. 进程打开的文件描述符
struct files_struct *files;
```

在内核中实际上是以一个数组的形式存在的, 我们大名鼎鼎的socket就是在这个位置存储的

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250811232713.png)

其中前三个打开的文件就是我们的标准输入, 标准输入, 标准错误, 这也是为什么这些std对应的数字是0,1,2

### 进程的命名空间

这块的内容其实是服务于容器技术的, 通过命名空间来提供可见性, 但是并不保证隔离性. 和CPP中的命名空间的概念差不多, 本质上还是提供可见性上的区别

```c
// 8. namespace
struct nsproxy *nsproxy;

struct nsproxy {
    atomic_t count;
    struct uts_namespace *uts_ns;
    struct ipc_namespace *ipc_ns;
    struct mnt_namespace *mnt_ns;
    struct pid_namespace *pid_ns;
    struct net			 *net_ns;
}
```

对于本机非容器环境来说, 就一个固定的命名空间

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/20250811233824.png)