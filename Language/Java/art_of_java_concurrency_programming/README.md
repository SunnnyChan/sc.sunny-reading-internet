# Java 并发编程的艺术
```md
By 方腾飞 魏鹏 程晓明
Reading Note Start : 2018-12-06
```

## Chapters

### [[01 - 并发编程的挑战]](01-challenge.md)
* [上下文切换]
* [死锁]
* [资源限制的挑战]

### [02 - Java 并发机制实现原理]
* [volatile 的应用](02-underlying-principle/02-01_volatile_application.md)
* [synchronized 的实现原理与应用](02-underlying-principle/02-02_synchronized.md)
* [原子操作的实现原理](02-underlying-principle/02-03_atomic_oprations.md)

### [03 - Java 内存模型]
* [内存模型基础](03-JMM/03-1-JMM.md)
* [重排序](03-JMM/03-2-reorder.md)
* [顺序一致性](03-JMM//03-3-sequential-consistency.md)
* [volatile 的内存语义](03-JMM/03-04-volatile-memory-semantics.md)
* [锁 的内存语义](03-JMM/03-05-lock-memory-semantics.md)
* [final 域的内存语义](03-JMM/03-06-final-memory-semantics.md)
* [happens-before](03-JMM/03-07-happens-before.md)
* [双重检查锁定和延迟初始化](03-JMM/03-08-recheck-lock-and-latency-init.md)
* [内存模型综述](03-JMM/03-09-summary.md)

### [04 - Java 并发编程基础]
* [线程](04-thread/04-01_thread.md)
* [启动和终止线程](04-thread/04-01_thread.md)
* [线程间通信](04-thread/04-02_thread_comunication.md)
* [线程应用实例](04-thread/04-03_thread_appliaction.md)

### [05 - Java 中的锁]
* [Lock 接口](05-lock_in_java/05-01_Lock-interface.md)
* [队列同步器]
* [重入锁]
* [读写锁]
* [LockSupport 工具]
* [Condition 接口]

### [06 - Java 并发容器和框架]
* [ConcurrentHashMap](06-concurrent-container-and-frame/06-01_ConcurrentHashMap.md)
* [ConcurrentLinkedQueue](06-concurrent-container-and-frame/06-02_ConcurrentLinkedQueue.md)
* [Java 中的阻塞队列](06-concurrent-container-and-frame/06-03_blocking-queue.md)
* [Fork/Join 框架](06-concurrent-container-and-frame/06-04_Fork-Join.md)

### [[07 - Java 中的13个原子操作类]](07-atomic-class.md)

### [08 - Java 中的并发工具类]
* [等待多线程完成的 CountDownLatch](01-JCU/08-01_CountDownLatch.md)
* [同步屏障 CyclicBarrier](01-JCU/08-02_CyclicBarrier.md)
* [控制并发线程数 Semaphore](01-JCU/08-03-Semaphore.md)
* [线程间交换数据 Exchanger](01-JCU/08-04_Exchanger.md)

### [09 - Java 中的线程池]

### [10 - Executor 框架]
* [Executor](10-executor/10-01_Executor.md)
* [ThreadPoolExecutor](10-executor/10-02_ThreadPoolExecutor.md)
* [ScheduledThreadPoolExecutor](10-executor/10-03_ScheduledThreadPoolExecutor.md)
* [FutureTask](10-executor/10-04_FutureTask.md)

### [11 - Java 并发编程实践]
* [生产者消费者模式](11-practice/11-01_Producer-and-Comsumer.md)
* [线上问题定位]
* [性能测试]
* [异步任务池]



## 相关参考
* [相关代码](https://github.com/soogbo/ArtConcurrentBook/tree/master/source/ArtConcurrentBook/src)