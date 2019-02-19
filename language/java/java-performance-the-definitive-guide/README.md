# Java 性能权威指南
> Java Preformance : The Definitive Guide
```md
By Scott Oaks
Reading Note Start : 2018-12-03 
```

## Chapters
### [01 - 导论]
* 全面的性能调优

### [[02 - 性能测试方法]](chapter/chapter-2_performance-test.md)
* 原则1：测试真实应用
* 原则2：理解批处理流逝时间、吞吐量和响应时间
* 原则3：用统计方法应对性能变化
* 原则4：尽早频繁测试

### [[03 - 工具]](chapter/chapter-3_tools.md)
* 操作系统的工具和分析
* Java 监控工具
* 性能分析工具
* Java 任务控制

### [04 - JIT]
* JIT 编译器：概览
* 调优入门：选择编译器类型
* Java 和 JIT 编译器版本
* 编译器中级调优
* 高级编译器调优
* 逆优化
* 分层编译级别

### [05 - GC 入门]
* [垃圾收集概述](chapter/chapter-5.1_GC.md)
* [GC 调优基础](chapter/chapter-5.2_GC_IM_basic.md)
* [垃圾回收工具](chapter/chapter-5.3_GC_tools.md)

### [06 - GC 算法]
* 理解 Throughput 收集器
* [理解 CMS 收集器](chapter/chapter-6.2_GC-CMS.md)
* [理解 G1 垃圾收集器](chapter/chapter-6.3_GC-G1.md)
* [高级调优](chapter/chapter-6.4_advance_IM.md)

### [07 - 堆内存最佳实践]
* [堆分析](chapter/chapter-7.1_heap-memory-analysis.md)
* [减少内存使用](chapter/chapter-7.2_reduce-memory-usage.md)
* [对象生命周期管理](chapter/chapter-7.3_object-lifecycle-mgt.md)

### [08 - 原生内存最佳实践]
* [内存占用](chapter/chapter-8_native-momery.md)
* 针对不同OS优化JVM

### [09 - 线程与同步的性能]
* 线程池 与 ThreadPoolExecutor
* ForkJoinPool
* 线程同步
* JVM 线程调优
* 监控线程和锁

### [10 - Java EE 性能调优]
* Web 容器的基本性能
* 线程池
* EJB 会话 Bean
* XML 和 JSON 处理
* 对象序列化

### [11 - 数据库性能最佳实践]
* JDBC
* JPA

### [12 - Java SE API 技巧]
* 缓冲式IO
* 类加载
* 随机数
* Java 原生接口
* 异常
* 字符串性能
* 日志
* Java 集合类API
* AggressiveOpts 标志
* Lambda 表达式和匿名类
* 流和过滤器的性能

### 参考
[Examples from the book](https://github.com/snattrass/java-performance-definitive-guide)
[Examples for the book](https://github.com/ScottOaks/JavaPerformanceTuning)