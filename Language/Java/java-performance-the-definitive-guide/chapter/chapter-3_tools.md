# Java 性能调优工具箱
> 如何获取并理解性能数据

## 操作系统的工具和分析
* CPU 使用率

* CPU 运行队列

* 磁盘使用率

* 网络使用率


## Java 监控工具
### JDK 工具
#### jcmd
```md
打印 Java 进程涉及的基本类、线程 和 VM信息。
```
* [详细使用指南](https://www.jianshu.com/p/388e35d8a09b)
#### jconsole
```md
JVM 活动的图形化视图，包括线程的使用、类的使用 和 GC 活动。
```
#### jhat
```md
读取内存转储堆，并有助于分析。这是事后使用的工具。
```
#### jmap
```md
提供内存转储堆和其他JVM内存使用的信息。
可以适用于脚本，但堆转储必须在事后分析工具中使用。
```
* [详细使用指南](https://www.cnblogs.com/kongzhongqijing/articles/3621163.html)
#### jinfo
```md
查看JVM的系统属性，可以动态设置一些系统属性。可适用于脚本。
```
* [详细使用指南](https://www.jianshu.com/p/ece32dacce64)
#### jstack
```md
转储进程的栈信息。可适用于脚本。
```
#### jstat
```md
提供 GC 和 类装载活动的信息，可适用于脚本。
```
#### jvisualvm
```md
监视JVM的GUI工具，可用来剖析运行的应用，分析JVM堆转储。
```

### 获取信息
#### 基本的 VM 信息
```md
JVM 工具可以提供JVM 进程的基本运行信息。
```
* JVM 运行时长
```sh
$ jcmd 8494 VM.uptime
8494:
247823.818 s
```
* 系统属性
```md
包括 通过 命令行 -D 标志设置的属性，应用动态添加的属性 和 JVM 默认属性。
```
```sh
$ jcmd 8494 VM.system_properties
8494:
#Tue Dec 04 11:58:46 CST 2018
java.runtime.name=Java(TM) SE Runtime Environment
sun.boot.library.path=/usr/local/jdk1.8.0_77/jre/lib/amd64
java.vm.version=25.77-b03
java.vm.vendor=Oracle Corporation
java.vendor.url=http\://java.oracle.com/
path.separator=\:
... ...
```
* JVM 版本
```sh
$ jcmd 8494 VM.version
8494:
Java HotSpot(TM) 64-Bit Server VM version 25.77-b03
JDK 8.0_77
```
* JVM 命令行
```sh
$ jcmd 8494 VM.command_line
8494:
VM Arguments:
jvm_args: -Xms1g -Xmx4g
java_command: com.didichuxing.report.application.ReportApplication server ./config/dps_report_rest.yml
java_class_path (initial): dps_report_rest-1.0-SNAPSHOT.jar
Launcher Type: SUN_STANDARD
```
* JVM 调优标志
* * 运行时调优标志值
```sh
$ jcmd 8494 VM.flags
8494:
-XX:CICompilerCount=15 -XX:InitialHeapSize=1073741824 -XX:MaxHeapSize=4294967296 -XX:MaxNewSize=1431306240 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=357564416 -XX:OldSize=716177408 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -X$
:+UseFastUnorderedTimeStamps -XX:+UseParallelGC
```

```sh
$ jcmd 8494 VM.flags -all
8494:
[Global flags]
    uintx AdaptiveSizeDecrementScaleFactor          = 4                                   {product}
    uintx AdaptiveSizeMajorGCDecayTimeScale         = 10                                  {product}
    uintx AdaptiveSizePausePolicy                   = 0                                   {product}
    uintx AdaptiveSizePolicyCollectionCostMargin    = 50                                  {product}
    ... ...
    uintx YoungPLABSize                             = 4096                                {product}
     bool ZeroTLAB                                  = false                               {product}
     intx hashCode                                  = 5                                   {product}
```
```md
列出 JVM 运行时内部所有的调优标志，大约几百个。
建议永远不要对其做修改，但诊断性能问题时，找出那些起作用的标志是有必要的。
```
* * 调优标志默认值
```md
* JVM 调优标志默认值
$ java  -XX:+PrintFlagsFinal -version
[Global flags]
     ... ...
     intx CICompilerCount                          := 15                                  {product}
     bool CICompilerCountPerCPU                     = true                                {product}
     ... ...
     intx BackEdgeThreshold                         = 100000                              {pd product}
     ... ...

输出中的 冒号表示非默认值。可能是因为：
1. 标志值直接在命令行指定
2. 其它标志简洁改变了该标志的值
3. JVM 自动优化计算出来的默认值

product 表示所有平台设置的默认值是一致的。
pd product 表示默认值是独立于平台的。
manageable 表示运行时值可以动态更改的标志。
C1 product 为编译器工程师提供诊断输出，帮助理解编译器正以什么方式运作。
```
* * jinfo
```md
jinfo 也可以查看调优标志值，好处是运行程序在运行时更改标志值。
但并不意味着JVM 一定会响应更改，比如GC算法行为的标志，在启动的时候使用，以决定GC的行为方式。
所以 只对 PrintFlagsFinal 输出中 标记为manageable调优标志有用。
```
* * * 显示所有标志值
```sh
$ jinfo -flags 8494
Attaching to process ID 8494, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.77-b03
Non-default VM flags: -XX:CICompilerCount=15 -XX:InitialHeapSize=1073741824 -XX:MaxHeapSize=4294967296 -XX:MaxNewSize=1431306240 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=357564416 -XX:OldSize=716177408 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC
Command line:  -Xms1g -Xmx4g
```
* * * 检查单个标志值
```md
* 开启
$ jinfo -flag +PrintGCDetails 34931
* 查看
$ jinfo -flag PrintGCDetails 34931
-XX:+PrintGCDetails
* 关闭
$ jinfo -flag -PrintGCDetails 34931
$ jinfo -flag PrintGCDetails 34931
-XX:-PrintGCDetails
```
#### 线程信息
```md
jconsole jvisualvm 可以实时显示应用中运行的线程数量。
jstack 可以获取栈信息。
```
```md
* jstack 每个线程栈输出
$ jstack  34931
2018-12-04 14:30:35
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.77-b03 mixed mode):

"qtp782689036-406086" #406086 prio=5 os_prio=0 tid=0x00007efd6c017000 nid=0x46a5 waiting on condition [0x00007efee1de2000]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000004c002c120> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
... ...

* jcmd 每个线程栈输出
$ jcmd 34931 Thread.print
```
> 具体可以参考 [Chapter 9](chapter-9_thread-sync.md)
#### 类信息
```md
jconsole jstat 提供已使用类的个数。
jstat 还能提供类编译的相关信息。
```
> 具体可以参考 [Chapter 12 监控类编译相关内容](chapter-12_java-SE-API.md)
#### 实时GC分析
```md
几乎所有的监控工具都能提供一些GV活动的信息。
jconsole 用实时图显示对的使用情况。
jcmd 可以执行 GC 操作。
jmap 可以打印堆的情况、永久代信息、创建堆转储。
```
> 具体可以参考 [Chapter 5](chapter-5_GC.md)
#### 事后堆转储
```md
jvisualvm GUI界面可以捕获堆转储，也可以用jmap 和 jcmd 生成。
堆转储是堆使用情况的快照，可以用不同的工具分析，包括 jvisualvm 和 jhat。

传统上第三方工具领先于JDK，可以参考 Chapter 7 使用的第三方工具 - Eclipse Memory Analyser Tool。
```
## 性能分析工具

### 采样分析器

### 探查分析器

### 阻塞方法 和 线程时间线

### 本地分析器

## Java 任务控制
```md
Java 7 和 Java 8 包含了 称为 JMC (Java Mission Control）的监控新特性。
JMC 需要商业许可才用可。

JMC 开启一个窗口显示当前机器上的JVM进程。
```
### Java 飞行记录器
```md
JFR (Java Flight Recorder)是JMC的关键特性。
JFR 数据是JVM的历史事件，可以用来诊断 JVM 的历史性能和操作。
```
### 开启 JFR

### 选择 JFR 事件
