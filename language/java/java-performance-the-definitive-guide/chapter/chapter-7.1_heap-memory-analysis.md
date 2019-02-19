# 堆内存最佳实践
```md
内存的使用有两个相互冲突的目标：
1. 一般规则是，有截止地创建对象并尽快丢弃。使用更少的内存，这是提升GC效率的最好方法。
  相反频繁地重建对象会导致整体性能变得更糟。
2. 然而，如果重用那些对象，程序性能可能会得到改善。
   这些对象包括 线程局部变量、特殊对象的引用以及对象池。
   重用意味着对象长期存活，且会影响GC，但如果能合理利用，整体性能会得到提升。

重要的是如何在两种方法之间做权衡。
```
## 堆分析
```md
大多数堆分析工具仅输出活跃对象（不包含下一次 Full GC 周期内被回收的对象）。
在某些情况下，这些工具会强制执行一次 Full GC来实现其功能，此时应用的行为会受影响。
而且工具也会消耗一定的时间和机器资源。
```
### 堆直方图
> 确认哪类对象消耗了大量内存
```sh
$ jcmd pid GC.class_histogram
```
```md
如果内存压力是由一些特定的对象类型引起的，利用直方图我们很快能看出端倪。
而直方图不需要做堆转储。
```
```sh
$ jcmd 20105 GC.class_histogram
20105:

 num     #instances         #bytes  class name
----------------------------------------------
   1:          6334        7603328  [B
   2:         13084        1098792  [C
   3:          3363         375232  java.lang.Class
   4:         12980         311520  java.lang.String
   5:          2911         172232  [Ljava.lang.Object;
   6:          5077         162464  java.util.concurrent.ConcurrentHashMap$Node
   7:          1692         148896  java.lang.reflect.Method
   8:          3679         147160  java.lang.ref.Finalizer
   9:          1819         101864  java.util.zip.ZipFile$ZipFileInflaterInputStream
  10:          1819         101864  java.util.zip.ZipFile$ZipFileInputStream
  11:          1167          96560  [Ljava.util.HashMap$Node;
  12:          2647          84704  java.util.HashMap$Node
  13:          4724          75584  java.lang.Object
  14:          1709          68360  java.util.LinkedHashMap$Entry
  15:          1341          63816  [I
  16:          1119          62664  java.util.LinkedHashMap
  17:          1946          47728  [Ljava.lang.Class;
  18:            55          46992  [Ljava.util.concurrent.ConcurrentHashMap$Node;
  19:           566          40752  java.lang.reflect.Field
  20:           918          36720  java.util.WeakHashMap$Entry
  21:           431          34480  java.lang.reflect.Constructor
  22:           718          34464  java.util.HashMap
  23:          1239          29736  java.util.ArrayList
  24:           559          29160  [Ljava.lang.String;
  25:          1139          27336  java.util.LinkedList$Node
  26:           834          26688  java.util.LinkedList
  27:           664          26560  java.lang.ref.SoftReference
  28:           906          21744  org.glassfish.jersey.server.internal.routing.CombinedMediaType$EffectiveMediaType
  29:           373          20888  java.lang.Class$ReflectionData
  30:           393          16808  [Ljava.lang.reflect.Method;
  31:            38          14288  java.lang.Thread
... ... 
```
```md
Klass 相关的对象往往接近顶端，他们是类加载的元数据对象。
顶端 字符数组([C) 和 String 也比较常见，它们是最长创建的Java对象。
字节数组([B) 和 Object数组（[Ljava.lang.Object）也常见，因为类加载器会将其数据保存到这些结构中。
```
* jmap -histo pid
```md
会得到类似输出，但包含会被回收的对象。
```
* jmap -histo:live pid
```md
强制执行一次GC
```
* 小结
```md
直方图较小，在自动化系统中为多次收集会很有帮助。
因为直方图也需要几秒钟，所以不要在性能测量稳定的状态下获得。
```
### 堆转储
```md
如果需要深度分析，就需要堆转储。很多工具可以连接到应用程序来生成转储文件。
```
* 从命令行生成
```sh
$ jcmd 20105 GC.heap_dump heap_dump.hprof
20105:
Heap dump file created

$ jmap -dump:live,file=heap_dump.prof 20105
Dumping heap to /home/xxx/heap_dump.prof ...
Heap dump file created
```
```md
jamp 指定 live 表示在dump之前会做一次 Full GC，jcmd 默认会执行。
如果希望包含 Full GC 对象，执行 $ jcmd 20105 GC.heap_dump heap_dump.hprof -all
```
* 分析
* * jhat
```md
最原始的堆分析工具，运行一个小型HTTP服务，通过一些列网页展示堆转储信息。
```
* * jvisualvm
```md
监视选项卡可以从一个运行中的程序获取堆转储文件，也可以打开dump出来的文件。
可以浏览堆，检查最大的保留对象，已经执行任意针对堆的查询。
```
* * mat
```md
开源EclipseLink内存分析工具（EclipseLink Memory Analyzer Tool， mat）
可以加载一个或多个堆转储文件分析。
能生成报告，建议那些可能存在问题的地方，也可以用于浏览堆，并对堆执行SQL的查询。
```
```md
第一遍分析一般涉及保留内存，一个对象的保留内存是指回收该对象可以释放出的内存。
```
* 浅对象大小、保留对象大小、深对象大小
```md
一个对象的浅大小指对象本身的大小。
如果该对象包含另一个对象的引用，4字节或8字节的引用会计算在内，但目标对象大小不会。
深大小则包含那些对象的大小。

深大小和保留大小区别在于存在共享的对象。保留大小不包含共享的对象大小。
```
* 堆的支配者
```md
保留了大量堆空间的对象一般被称作堆的支配者。
我们需要减少堆的支配者对象的创建，减少保留这类对象的时间，简化其对象图，或者将对象变小。

因为程序可能共享，所以有时候需要做一些侦查性工作。
共享对象不会计算在任何其它对象的保留集内，因为单独释放一个对象不会释放共享对象。
此外，最大的保留大小往往是我们无法控制的类加载器带来的。
```
* 示例
```md
* 查看保留内存
查看有多个实例的情况，且保留了相当数量的内存。
一般情况下，内存可能会被共享，所以从保留堆可能看不任何明显的东西。

* 查看直方图
直方图将同一类型的对象聚合到一起，可能会更容易看出来问题。
此时找到GC根，GC根保存一些指向问题中对象的静态和全局引用。
它们通常来自系统或Bootstarp类路径下加载的某个类的静态变量，
包括 Thread 类 和 所有的活跃线程；线程保留对象，或是通过其线程局部变量，
或是通过目标Runnable 对象（或者在存在Thread 子类的情况下，通过子类中包含的任何其它引用）来引用。

* 追溯对象引用
某些情况下，知道目标对象的GC根是有用的，但如果是多个指向该对象的引用，那么会有多个GC根。
引用有可能会随着追根溯源的过程爆炸性增长。

检查并找出对象被共享的最下面的一点可能更有效。
实现方法是检查对象及指向该对象的引用，然后跟踪这些引用，直到识别出重复的路径。

如果应用中主要数据被建模为String对象，分析会更加困难，因为堆转储中会有数十万其它字符串。
一般来说，要从集合类对象（如 HashMap）入手，而不记录项（HashMap$Entry），并且要寻找最大的集合。
```
* 小结
```md
了解哪些对象正在消耗内存，是优化的第一步。
直方图对于识别某一种特定类型对象引发的内存问题快速且方便。
堆转储分析是追踪内存使用的最强大的技术，不过要用好，需要一定的耐心和努力。
```
### 内存溢出错误
* OOM 错误 发生场景
```md
JVM 没有原生内存可用
永久代（Java 7）和元空间（Java 8）内存不足
堆内存本身不足 —— 对给定的堆空间，应用活跃对象太多
JVM 执行GC 耗时太多

后面两种情况涉及Java堆本身，更为常见。
```
* 原生内存不足
```md
当堆进行分析解决不了问题，需要看看错误中提到的是何种原生内存问题。
如：下面的提示说明线程栈的原生内存消耗尽了：
Exception in thread "main" java.lang.OutOfMemory:
unable to create new native thread
```
> 具体可以参考 [Chapter 8](chapter-8_native-momery.md)
* 永久代或元空间内存不足
```md
错误发生的原因是永久代或元空间原生内存满了。

根源可能有两种情况：
1. 应用使用的类太多，超出了永久代的默认容纳范围，需要增加永久代空间大小。
2. 涉及类加载器的内存泄漏。这种情况经常出现在Java EE 的应用服务器中。
  部署到应用服务器的每个应用都运行在自己的类加载器中（提供了隔离）。
  如果应用会创建并丢弃大量的类加载器，一定要谨慎，确保类加载器能正常丢弃
  （尤其是确保没有线程将其上下文加载器设置成一个临时的类加载器）

  要调试这种情况，在直方图中，找ClassLoader类的所有实例，然后跟踪它们的GC根，
  看哪些对象还保留着对它们的引用。

  在Java 8中，如果元空间满会输出下面的错误信息：
  Exception in thread "main" java.lang.OutOfMemoryError: Metaspace

  在Java 7中，错误信息如下：
  Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
```
> [PermGen与Metaspace](https://segmentfault.com/a/1190000012577387)
* 堆内存不足
```md
* 错误消息：
 Exception in thread "main" java.lang.OutOfMemoryError: Java heap space

* 分析
发生的情况和 永久代 的情况类似：
可能应用需要更大的堆空间，也可能是存在内存泄漏。

如果应用存在内存泄漏，可以隔几分钟转储一次，mat 有一个选项来计算两个堆中的直方图的差别。

* 自动转储堆
-XX:HeaoDumpOnOutOfMemoryError
在抛出OOM时自动创建堆转储，默认关闭。

-XX:HeapDumpPath=<path>
指定堆转储文件写入位置，可以指定目录，也可以指定文件名。
默认会在应用的当前工作目录下生成 java_pid.hprof文件。

-XX:+HeapDumpAfterFullGC 或 -XX:+HeapDumpBeforeFullGC
Full GC 后或前 创建堆转储文件。

如果应用因为堆空间的原因，不可预测地抛出OOM，请尝试打开这些标志。

* 实践
集合类是导致内存泄漏的最常见原因：应用向集合中插入条目，但从不释放它们。
要克服这种情况，好的办法是主动将不再使用的条目从集合中删除。

作为一种选择 可以使用 软应用 或 弱引用的集合，
当在应用中已经不存在对这些条目的任何引用时，集合会自动丢弃它们。
不过这样的集合是有代价的。
```

* 达到 GC 开销限制
```md
JVM 认为在GC上花费太多的时间，也会抛出 OnOutOfMemoryError。
 Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded

* 抛出该错误必须满足的条件：
1. Full GC 的时间超出了 -XX:GCTimeLimit=N 指定的值。
  默认值是 98（即 如果98%的时间花在额GC上）
2. 一次 Full GC 回收的内存数量 少于 -XX:GCHeapFreeLimit=N 指定的值。
  默认值是 2（即 释放的内存不足堆的 2%）
3. 上面两个条件 连续 5次 Full GC 都成立（该值无法调整）
4. -XX:+UseGCOverHead-Limit 标志值为true（默认如此）

* 注意
1. 当 Full GC 多次连续执行 花费时间超过 98%，但可能释放的堆内存大于 2%，
  此时可以考虑修改 -XX:GCHeapFreeLimit的值。
2. 如果前两个条件连续4次成立，作为释放内存的最后一搏，
  JVM中所有的软引用会在第5次Full GC之前被释放，这往往会防止错误发生。
```