---
title: JUC-线程池-ThreadPoolExecutor类详解
tag: ["Java", "JUC", "线程池"]
categories: ["Java", "JUC", "线程池"]
type: Java
---

# 线程池-ThreadPoolExecutor类详解

## ThreadPoolExecutor的构造方法

```java
public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
            Executors.defaultThreadFactory(), defaultHandler);
}
```
- corePoolSize: 核心线程数, 线程池中即使没有任务也要保持存活的最大线程数量
- maximumPoolSize: 线程池中允许的最大线程数量
- keepAliveTime: 线程池中超过核心线程数的空闲线程存活时间
- unit: keepAliveTime的时间单位
- workQueue: 任务队列, 用于存放待执行的任务
- threadFactory: 线程工厂, 用于创建新线程
- handler: 拒绝策略, 当任务无法被执行时的处理策略

### 拒绝策略

JDK提供了四种默认的拒绝策略

- AbortPolicy: 将任务丢弃并抛出一个`RejectedExecutionException`异常, 默认的拒绝策略

```java
    public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                    " rejected from " +
                                                    e.toString());
        }
    }
```

- DiscardPolicy: 将任务直接丢弃

```java
    public static class DiscardPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardPolicy}.
         */
        public discardpolicy() { }

        /**
         * does nothing, which has the effect of discarding task r.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedexecution(runnable r, threadpoolexecutor e) {
        }
    }
```

- CallerRunPolicy: 直接在试图创建并执行任务的calling Thread线程执行这个任务

```java
    public static class CallerRunsPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code CallerRunsPolicy}.
         */
        public CallerRunsPolicy() { }

        /**
         * Executes task r in the caller's thread, unless the executor
         * has been shut down, in which case the task is discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }
```

- DiscardOldestPolicy: 将阻塞队列的最后一个任务丢弃, 然后重新执行这个任务 

```java
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardOldestPolicy} for the given executor.
         */
        public DiscardOldestPolicy() { }

        /**
         * Obtains and ignores the next task that the executor
         * would otherwise execute, if one is immediately available,
         * and then retries execution of task r, unless the executor
         * is shut down, in which case task r is instead discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```

## 核心字段

源码
```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
```

- ctl: Control线程控制信号, 是一个原子类, 前三位是线程池的状态位, 后29位是线程池中的线程数量
  - ` int ctlOf(int rs, int wc) { return rs | wc; }`方法构建ctl
    - rs是RunState 线程池状态
    - wc是WorkerCount 工人数量也就是线程池中的线程的数量
  - `int runStateOf(int c)     { return c & ~COUNT_MASK; }`: 获取线程池状态
  - `int workerCountOf(int c)  { return c & COUNT_MASK; }`: 获取工人数量
- COUNT_BITS: 这个参数 == 29, 是获取状态位和设置状态位需要移动的位数
- CONUT_MASK: 前29bit都是1, c & COUNT_MASK来获取到wc
- RUNNING: 线程池的正常工作状态, 线程池可以接受新的任务并且会处理等待队列中的任务
- SHUTDOWN: 调用`shutdown()`方法的时候, 线程池进入到这个状态
  - 线程池不再接受新的任务
  - 继续执行等待队列中的已经存在的任务和正在执行的任务
- STOP: 调用`shutdownNow()`方法
  - 不接受新的任务
  - 不处理队列中的任务
  - 尝试中断正在执行的任务
- TIDYING: 一种过渡状态, 在满足以下条件的时候进入
  - 所有的任务都已经终止
  - 工作线程的数量是0
  - 队列为空的时候进入到这个状态以后, 会执行 `terminate()`钩子方法
- TERMINATED: 线程池关闭, 所有的资源都得到释放

![](https://cdn.jsdelivr.net/gh/fang-tech/Pic/img/202507231701892.png)

## 任务的执行

### execute方法

源码及解析

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        // 1. 如果现在的线程数量 < 核心线程数量
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            // 增加核心线程数量
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 2. 现在的线程数量 > 核心线程数量
        // 将任务添加到等待队列中
        if (isRunning(c) && workQueue.offer(command)) {
            // 双重检查, 因为进入到if块以后可能状态线程池的状态发生了变化
            int recheck = ctl.get();
            // 如果双重检查的时候发现线程池的状态不再是RUNNING, 移除任务
            // 并执行reject回调方法
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 执行到这个位置的时候, 逻辑上核心线程已经满了, 但是是可能出现其他的线程因为某些错误死掉的情况, wc实际记录的是还存活着的线程, 这个时候我们就需要创建一个非核心线程来执行
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 3. 等待队列也已经满了, 尝试创建非核心队列执行任务, 如果创建失败
        // 说明线程数量已经超过了最大线程数量, 或者线程池的状态不允许创建新的线程了
        else if (!addWorker(command, false))
            // 执行reject方法
            reject(command);
    }
```

线程池类执行任务的顺序是
1. 先尝试将任务交给核心线程, 如果核心线程的数量 < 最大核心线程数量, 创建新的核心线程执行任务
2. 核心线程的数量超过了最大线程, 尝试将任务添加到等待队列里面
3. 等待队列也已经满了, 创建一个非核心线程执行任务, 如果创建失败, 说明线程状态不是RUNNING或者线程数量已经超过了最大的数量, 这个时候执行reject方法

> 原文: 
> Proceed in 3 steps:
> 1. If fewer than corePoolSize threads are running, try to
> start a new thread with the given command as its first
> task.  The call to addWorker atomically checks runState and
> workerCount, and so prevents false alarms that would add
> threads when it shouldn't, by returning false.
> 
> 2. If a task can be successfully queued, then we still need
> to double-check whether we should have added a thread
> (because existing ones died since last checking) or that
> the pool shut down since entry into this method. So we
> recheck state and if necessary roll back the enqueuing if
> stopped, or start a new thread if there are none.
> 
> 3. If we cannot queue task, then we try to add a new
> thread.  If it fails, we know we are shut down or saturated
> and so reject the task.

### addWorker

方法比较长, 分成两部分解读

方法签名: `private boolean addWorker(Runnable firstTask, boolean core)`
- core: 添加的线程是不是核心线程

- 增加线程的数量, 并没有真的增加线程

```java
    retry:
    for (int c = ctl.get();;) {
        // 如果线程池的状态至少是SHUTDOWN, 这个时候我们不能再添加新的任务
        // 如果状态是STOP, 说明这个时候不能再添加任务了, return false
        // 如果状态是SHUTDOWN, 但是传入的任务不是null, 也返回null, 因为SHUTSOWN状态只能从任务队列中消费任务
        // 如果传入的任务是空, 但是任务队列也是空的, 这个时候没有要消费的任务了, retuen false
        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP)
                || firstTask != null
                || workQueue.isEmpty()))
            return false;

        // CAS增加worker的数量
        for (;;) {
            // 超过线程数量的限制, 不能再添加worker了
            if (workerCountOf(c)
                >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                return false; 
            // CAS成功增加worker的数量, 到下一步增加Worker
            if (compareAndIncrementWorkerCount(c))
                break retry; 
            c = ctl.get();  // CAS失败, 重新获取worker数量
            if (runStateAtLeast(c, SHUTDOWN))
                continue retry;
            // CAS失败, 说明在这个for中的CAS之前worker的数量发生了变化, CAS尝试添加线程
        }
    }
```


- 添加Worker, 也就是真的添加工作线程

Worker是一个继承了AQS, 实现了Runnable的类

```java
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        // w.thread是在构造方法中使用线程工厂创建的
        // this.thread = getThreadFactory().newThread(this);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            // 加锁保护HashSet的线程安全
            // largestPoolSize 统计信息的安全
            // 在双重检查时候, ws不会发生变化
            mainLock.lock();
            try {
                // 用于双重检查
                // 如果在获取锁以前, 线程池shutdown了
                int c = ctl.get();

                // 状态是RUNNING, 或者状态是SHUTDOWN并且任务是空
                // 前者正常情况, 后者是在创建非核心线程
                if (isRunning(c) ||
                    (runStateLessThan(c, STOP) && firstTask == null)) {
                    // 创建线程失败
                    if (t.getState() != Thread.State.NEW)
                        throw new IllegalThreadStateException();
                    // 创建成功, 将worker添加到线程池中
                    workers.add(w);
                    workerAdded = true;
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                }
            } finally {
                mainLock.unlock();
            }
            // 启动worker的任务
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 针对创建失败的Worker的处理
        if (! workerStarted)
            // 将这个worker从hashSet中移除
            addWorkerFailed(w);
    }
    return workerStarted;
```

### Worker类的构造方法


```java
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }
```

### runWorker方法

Worker类的run方法实现内部是直接调用runWorker(Worker w) 方法

```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock();
        boolean completedAbruptly = true;
        try {
            // 工作者线程不断从队列中尝试获取Task然后执行
            // 优先执行被分配的first Task
            while (task != null || (task = getTask()) != null) {
                // 加锁确保任务执行的时候不会被shutdown中断
                w.lock();
                // 先检查线程池状态 >= STOP, 是的话直接设置线程中断
                // 再次检查, 在线程已被中断, 线程池 >= STOP的时候
                    // 清除线程的中断标志, 然后设置线程被中断了
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    try {
                        // 核心执行内容, 执行task
                        task.run();
                        afterExecute(task, null);
                    } catch (Throwable ex) {
                        afterExecute(task, ex);
                        throw ex;
                    }
                } finally {
                    // 扫尾工作
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // 核心线程在RUNNIN步骤是不会走到这一步的, 
            // 因为会在getTask的过程中阻塞获取任务
            processWorkerExit(w, completedAbruptly);
        }
    }
```

### getTask方法

```java
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();

            // 在线程池 >= STOP, 等待队列是空的时候
            // 工人的数量--, 并直接返回null, 这种情况是getTask失败的情况
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
                // workerCount--
                decrementWorkerCount();
                return null;
            }

            // 运行到这里的时候, 说明运行状态是RUNNING
            // 或者 == SHUTDOWN, 或者 >= STOP 但是等待队列不是空

            int wc = workerCountOf(c);

            // 需不需要处理线程存活时间超时的情况
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            // 这里是用于处理当前的Worker是不是要销毁
            // 如果Worker的数量超过了最大线程池的大小, 减少多余的worker
            // 当前线程需要考虑超时, 并且上一次获取任务时发生了超时, 这种情况下worker也应该被回收以节省资源
            if ((wc > maximumPoolSize || (timed && timedOut))
                // 保证不会过度销毁Worker, Worker数量至少大于1
                // 或者等待队列中没有任务了, 这种情况也能销毁当前Worker
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    // 如果是非核心线程就会获取任务就会有超时设置
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    // 成功获取到了任务
                    return r;
                // 超时了, 在下一次循环中, 就会因为这个timeout = true
                // 而导致这个非核心线程被销毁
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

> 到这里我们能简单总结下一个Worker的生命周期
>
> 对于任何Worker, 会在runWorker方法中不断循环获取任务执行任务
> 一旦没有获取到任务, worker就会被移除
>
> 对于核心线程worker, 会在获取任务getTask()上一直阻塞直到获取任务
> 对于非核心线程worker, 在getTask上只会阻塞这个worker存活时间
> 超过这个时间, 就会在getTask的下一次循环中workerCOunt--返回null, 
> 然后结束runWorker中while循环, 然后将这个worker销毁

## 任务的提交

### submit方法

ThreadPoolExecutor类是继承自AbstractExecutorService的. 其中的submit方法也是在这个抽象类中实现的

```java
    public Future<?> submit(Runnable task) {
            if (task == null) throw new NullPointerException();
            // 通过submit方法提交的Callable任务会被封装成一个FutureTask对象
            // 普通的Runnable接口类, 也会被封装成FutureTask对象
            RunnableFuture<Void> ftask = newTaskFor(task, null);
            execute(ftask);
            return ftask;
        }
```

而execute就是我们第一个讲解的任务的执行的核心部分了

这里线程池的设计我们能看到是使用了模板模式

## 任务的关闭

### shutdown方法

将所有的正在阻塞获取任务的空闲线程的状态变成interrupt, 来释放没有在

怎么实现的SHUTDOWM状态的语义: 
  - 线程池不再接受新的任务
    - 在addWorker的时候, 如果是addWorker(command, true/false)形式都会返回false
  - 继续执行等待队列中的已经存在的任务和正在执行的任务
    - 只会通过interrupt唤醒没有在执行任务在阻塞获取Task的worker, 并且会删除这个空闲的worker
    - 不会影响正在执行的任务, 也不会影响在等待队列中还有任务的时候

怎么从SHUTDOWN一步一步变成的TERMINATE

1. 在execute中会直接reject新的任务
2. SHUTDOWN状态下并且队列为空的时候, 也就是开始出现空闲的worker的时候, 会在getTask方法返回null
3. 因为getTask方法返回了null, 触发了runWorker方法中的销毁worker, 并tryTerminate()
4. 在tryTerminate中再interrupt下一个worker, 这样渐进式将所有的worker都销毁

```java
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);
            // 将所有没有执行任务的worker打上中断状态
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }

    private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }

    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                // 成功获取到了worker的lock, 
                // 而worker的lock只会在runWorker方法类里面被获取走
                // 说明这个worker是一个空闲的worker
                // 通过中断来唤醒阻塞的worker来去检查是不是STOP了
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```

### shutdownNow方法

shutdownNow会为所有的线程都打上interrupt状态

- STOP: 调用`shutdownNow()`方法
  - 不接受新的任务
    - 同SHUTDOWN
  - 不处理队列中的任务
    - 会将队列清空并返回
  - 尝试中断正在执行的任务
    - 如果方法尝试执行, 但是还没有执行的时候, 也就是worker刚获取到下一轮的task的时候, 会因为状态是STOP, getTask() = null, 进入到销毁worker的过程

```java
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }

    private void interruptWorkers() {
        // assert mainLock.isHeldByCurrentThread();
        for (Worker w : workers)
            w.interruptIfStarted();
    }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
```