# 并发数据结构
CopyOnWriteArrayList
---
通过 COW 方式实现

BlockingQueue
---
咱这里以 ArrayBlockingQueue 为例。
*put()* *take()* 是可阻塞接口；*offer()* *pull()* 是定时可阻塞接口。

实现是用的内置锁 *lock* 中 *AQS* 的 *await()* *signal()*。

Latch 闭锁
---
让线程等在这里。调用接口减少计数，为0时唤醒所有等待线程。

FutureTask
---
一个把 *Future* 跟 *Callable* 的结合，鬼知道为什么要这么设计。在 *Executor* 中经常被使用。 *get()* 在任务结束前将会阻塞。JDK8 之前用 AQS 实现，现在改成用自己维护的 state 字段，用 CAS 来设置。因为它声称 AQS 版本会有取消相关的竞态。

Semaphore
---
和 AQS 本身最接近的。初始化资源数，耗尽时等待。

Barrier
---
初始化所需线程数量，当线程到齐时全部释放并执行一个回调 Runnable。

# 线程池 ThreadPoolExecutor
实现 *Executor* 接口，把丢进去的 *Runnable* 执行起来。
- 如果当前线程池中的线程数目 < *corePoolSize*，则每来一个任务，就会创建一个线程去执行这个任务
- 如果当前线程池中的线程数目 >= *corePoolSize*，则每来一个任务，会尝试将其添加到任务缓存队列当中。若队列满，则会尝试创建新的线程去执行这个任务。这里队列就是一个 *BlockingQueue*，大小初始化时可指定。
- 如果当前线程池中的线程数目达到 *maximumPoolSize* ，则会采取任务拒绝策略进行处理；
- 如果线程池中的线程数量 > *corePoolSize* 时，如果某线程空闲时间超过keepAliveTime，线程将被终止。
- 注意，JDK7 即使是 <= *corePoolSize*，线程仍会终止，不过会立马起个新的。参看 *processWorkerExit()*。

关闭 *Executor* 时，*shutDown()* 是清空队列后关闭，而 *shutDownNow()* 是立即关闭线程。在 *shutDown* 的情况，worker 获取不到任务就自己停了；在 *shutDownNow()* 的情况，直接调用所有 worker 的 *interrupt()*.

# 并发相关基础方法

中断相关
---
interrupt()：设置中断状态，并不能强制让其他线程停下。不过若处于阻塞状态，会使其抛出异常。注意，如果是 *wait()* ，中断后还需要等待锁。
interrupted()：查询中断标志并清空它。
isInterrupted()：查询中断标志但是不清空。

# 提升并发性能的原则
- 延迟优化，并以测试为基准确定优化效果。
- 多线程将引入 **线程切换 缓存失效** 的代价，并增加程序的复杂度。注意评估其效果。
- 多线程需要同步（锁和 volatile）。同步将会 **限制重排 并 迫使程序刷新缓存，造成缓存失效并增加 CPU 之间的通信量**。
- 应当减小锁的竞争，有以下几种努力方向：
    1. 缩减锁的范围，只锁必要的逻辑，让加锁的时间尽可能少。
    2. 减小锁的粒度，不需要同步的情况下拆解成一个对象一个锁。
    3. 锁分段。一个对象的锁甚至还能分段。但某些操作可能要求收集很多锁。
    4. 换用 Atomic 变量，或使用读写锁。