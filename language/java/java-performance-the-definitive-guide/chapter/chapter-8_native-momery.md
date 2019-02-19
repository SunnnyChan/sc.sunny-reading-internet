# 原生内存的最佳实践
```md
除堆以外，JVM 还会分配大量的原生内存（或者说操作系统内存）。
堆的配置以及堆如何与OS的原生内存交互，是影响应用程序整体性能的另一个重要因素。
```
## 内存占用
```md
JVM中堆消耗内存最多，同时JVM 也会为内部操作分配一些内存。
应用也可以分配原生内存：JNI调用malloc()，使用New I/O，即NIO时。

JVM 使用的原生内存和堆内存的总量，是一个应用总的内存占用（Footprint）。
从OS视角看，总的内存占用是性能的关键。

* 注意：
部分原生内存只是在启动时使用以下（如与加载 classpath 下的JAR 文件相关的内存），
如果这部分内存被交换 出去，我们未必会注意到。

有时候，一个 Java 进程使用的原生内存会与系统中的其它Java 进程共享，
还有少部分内存会与系统中的其它类型的进程共享。
不过多数情况下，为优化性能，我们希望确保所有 Java 进程总的内存占用不超过机器的物理内存
（加之 可能还要为其它应用保留一些内存）
```
### 测量内存占用
* 已分配内存 和 保留内存
```md
* -Xms512m -Xmx2048m 参数
JVM 告诉OS，它的堆可能需要多达 2G 的内存，所以会保留这么多内存，
OS 承诺，当JVM 需要这些内存时，是可以获取到的。

最初分配的内存仍然只有 512M，而且这就是堆实际用到的全部内存。
这些实际分配的内存就是 提交内存（或者说 已分配内存）
提交内存的量会随着堆的调整而波动，特别是提交内存会随着堆的增加而增加。

* 超量保留会不会有问题？
考察性能时，只有提交内存才有价值，绝对不会因为保留了太多内存出现性能问题。
不过有时候还是要确认JVM没有保留太多内存，32位JVM尤其如此。
64位JVM没有进程空间大小的限制，只受限于机器的虚拟内存总量。

凡事都有两面，给JVM 内部结构多分配些空间，让JVM优化其使用，这样比较方便，
但未必总是可行的。
```
* 线程栈
```md
JVM 几乎所有重要的内存区域中，会随着程序的运行 逐步增长。
线程栈是例外，是创建时全部分配的。
JVM每次创建线程，OS会分配一些原生内存来保存线程栈，向进程提交更多内存（至少要等到线程退出）。
```

* RSS
```md
一个进程实际的内存占用，可以使用OS工具报告的 进程驻留集（Resident Set Size，RSS） 大小来估算。
不过有两个不够精确之处：
1. 在JVM 和系统其他进程之间有些在OS层面共享的页面（共享库的text部分），会被计算在每个进程的RSS中。
2. 随时可能出现，一个进程提交内存多于实际调入的页面。

在 新的 Linux 内核中，PSS 取代了RSS，去掉了和其他进程共享的数据。
```

### 内存占用最小化
```md
* 堆
堆可能只占总内存占用的 50%到 60%。
可以将堆的最大值设置为一个较小的值（或者调整GC调优参数，比如控制堆不会被完全占满），
以限制进程的内存占用。

* 线程栈
线程栈非常大，特别是64位JVM。
参考 Chapter 9

* 代码缓存
代码缓存使用原生内存来保存编译后的代码。
```
### 原生 NIO 缓冲区
```md
如果 NIO 字节缓冲区是通过 allocateDirect()方法分配的，也会分配原生内存。
字节缓冲区的重要性在于它们支持原生代码 和 Java 代码在不复制的情况下共享数据。
最常见的是用户 文件系统和套接字操作的缓冲区。

把数据写入一个原生NIO缓冲区，在发送给通道（比如，文件或套接字），
不需要JVM 和 用于传输数据的C库之间复制数据，而使用堆字节缓冲区，则必须复制。

调用 allocateDirect() 是昂贵的，所以应尽可能重用直接字节缓冲区。
理想情况是，线程是独立的，而每个线程持有一个直接字节缓冲区作为线程局部变量。

当有很对线程需要大小不同的缓冲区，有时可能会消耗过多的原生内存，
因为每个线程的缓冲区最终可能会达到最大值，这时直接字节缓冲区的对象池更有用。
```
* 字节缓冲区切割管理
```md
应用可以分配一个大的直接字节缓冲区，然后每个请求使用ByteBuffer类的slice()方法从中分配一部分。

一个问题是，如果不能保证每次分配相同的大小，字节缓冲区可能会碎片化，
和堆不同的是，字节缓冲区的不同片段是无法严肃哦的，
所以只有当所有片段大小的相同时，以上的方案才好用。
```
* -XX:MaxDirectMemorySize
```md
直接字节缓冲区分配的内存总量通过 -XX:MaxDirectMemorySize 标志来指定。
从Java 7 开始，默认值 为0，意味着没有限制（当然受制于地址空间大小）。
```
* 小结
```md
从调优角度看，要控制JVM的内存占用，可以限制用于 直接字节缓冲区、线程栈、代码缓存的原生内存使用量。
```
### 原生内存跟踪
```md
原生内存跟踪(Native Memory Tracking, NMT)默认是关闭的。
```
```sh
$ jcmd  34931  VM.native_memory summary
34931:
Native memory tracking is not enabled
```
* -XX:NativeMemoryTracking
```md
从Java 8开始，可以通过 -XX:NativeMemoryTracking=off|summary|detail 来开启。
```
* * -XX:NativeMemoryTracking=summary
```sh
$ jcmd  44609  VM.native_memory summary
44609:

Native Memory Tracking:

Total: reserved=14543323KB, committed=4839151KB
-                 Java Heap (reserved=12582912KB, committed=4194304KB)
                            (mmap: reserved=12582912KB, committed=4194304KB)
                      # 用于保存类的元数据的原生内存
-                     Class (reserved=1098028KB, committed=51628KB) 
                            (classes #3135)
                            (malloc=35116KB #1979)
                            (mmap: reserved=1062912KB, committed=16512KB)
                    # 70 个线程栈分配的空间
-                    Thread (reserved=71229KB, committed=71229KB)
                            (thread #70)
                            (stack: reserved=70932KB, committed=70932KB)
                            (malloc=216KB #380)
                            (arena=81KB #138)
                      # JIT的代码缓存
-                      Code (reserved=250518KB, committed=8654KB)
                            (malloc=918KB #2061)
                            (mmap: reserved=249600KB, committed=7736KB)
                        # GC 算法处理 使用的一些对外内存
-                        GC (reserved=498924KB, committed=471624KB)
                            (malloc=39208KB #217)
                            (mmap: reserved=459716KB, committed=432416KB)
                  #编译器自身操作使用
-                  Compiler (reserved=141KB, committed=141KB)
                            (malloc=10KB #72)
                            (arena=131KB #3)

-                  Internal (reserved=35999KB, committed=35999KB)
                            (malloc=35967KB #5089)
                            (mmap: reserved=32KB, committed=32KB)
                      # 保留字符串（Interned String）的引用于符号表引用
-                    Symbol (reserved=4718KB, committed=4718KB)
                            (malloc=3687KB #30868)
                            (arena=1031KB #1)
      # NMT 本身操作也需要一些空间
-    Native Memory Tracking (reserved=649KB, committed=649KB)
                            (malloc=9KB #100)
                            (tracking overhead=640KB)

-               Arena Chunk (reserved=203KB, committed=203KB)
                            (malloc=203KB)
```
* * -XX:NativeMemoryTracking=detail
```md
包括整个内存空间的一个映射。
```
```sh
$ jcmd 3844 VM.native_memory detail
开始部分输出与 summary 一致。
Virtual memory map:

[0x00000004c0000000 - 0x00000007c0000000] reserved 12582912KB for Java Heap from
    [0x00007ff6f0a3fad2] ReservedSpace::initialize(unsigned long, unsigned long, bool, char*, unsigned long, bool)+0xc2
    [0x00007ff6f0a404ae] ReservedHeapSpace::ReservedHeapSpace(unsigned long, unsigned long, bool, char*)+0x6e
    [0x00007ff6f0a0da0b] Universe::reserve_heap(unsigned long, unsigned long)+0x8b
    [0x00007ff6f08c7874] ParallelScavengeHeap::initialize()+0x84

        [0x00000004c0000000 - 0x000000056ab00000] committed 2796544KB from
            [0x00007ff6f0912543] PSVirtualSpace::expand_by(unsigned long)+0x53
            [0x00007ff6f0902687] PSOldGen::initialize(ReservedSpace, unsigned long, char const*, int)+0xb7
            [0x00007ff6f026127a] AdjoiningGenerations::AdjoiningGenerations(ReservedSpace, GenerationSizer*, unsigned long)+0x39a
            [0x00007ff6f08c79c6] ParallelScavengeHeap::initialize()+0x1d6

        [0x00000006c0000000 - 0x0000000715500000] committed 1397760KB from
            [0x00007ff6f0912543] PSVirtualSpace::expand_by(unsigned long)+0x53
            [0x00007ff6f0913505] PSYoungGen::initialize_virtual_space(ReservedSpace, unsigned long)+0x75
            [0x00007ff6f0913e6e] PSYoungGen::initialize(ReservedSpace, unsigned long)+0x3e
            [0x00007ff6f0261225] AdjoiningGenerations::AdjoiningGenerations(ReservedSpace, GenerationSizer*, unsigned long)+0x345
... ...
```
```md
保留空间放在initialize()中，分配在 expand_by()中进行。
```
* -XX:NativeMemoryTracking
```md
如果启动了 -XX:+PrintNMTStatistics 参数（默认false），会在程序退出时打印原生内存的分配信息。 
```
* NMT 提供了两类关键信息
```md
* 总提交大小
是进程实际将要消耗的实际物理内存量，这个值应与RSS很接近。
实际测量值存在一个问题，有些内存虽然已经提交，但其页面被置换出去了，RSS不会计算在内。
实际上如果RSS小于一提交内存，就通常表明OS很难将JVM的所有信息放入内存中。

* 每部分的提交大小
当需要调优 堆、代码缓存】元空间等不同部分的最大值时，了解它们的实际使用会非常有用。
超量分配通常指影响内存的保留，不过有些情况下，保留内存也很重要，
而NMT 可以帮助我们跟踪那些可以再缩减的情况。
```
* 跟踪内存分配随时间的变化
* * 记录内存分配情况，作为基线
```sh
$ jcmd 20105 VM.native_memory baseline
20105:
Baseline succeeded
```
* * 比较当前内存分配情况与基线差别
```sh
$ jcmd 20105 VM.native_memory summary.diff
20105:

Native Memory Tracking:
// 与基线相比较，保留内存增加了 +1035KB
Total: reserved=14543328KB +1035KB, committed=4839156KB +1035KB

-                 Java Heap (reserved=12582912KB, committed=4194304KB)
                            (mmap: reserved=12582912KB, committed=4194304KB)

-                     Class (reserved=1098028KB, committed=51628KB)
                            (classes #3135)
                            (malloc=35116KB #1972 +1)
                            (mmap: reserved=1062912KB, committed=16512KB)

-                    Thread (reserved=71229KB +1032KB, committed=71229KB +1032KB)
                            (thread #70 +1)
                            (stack: reserved=70932KB +1028KB, committed=70932KB +1028KB)
                            (malloc=216KB +3KB #380 +5)
                            (arena=81KB +1 #138 +2)

-                      Code (reserved=250514KB, committed=8650KB)
                            (malloc=914KB #2062)
                            (mmap: reserved=249600KB, committed=7736KB)

-                        GC (reserved=498924KB, committed=471624KB)
                            (malloc=39208KB #217)
                            (mmap: reserved=459716KB, committed=432416KB)

-                  Compiler (reserved=141KB, committed=141KB)
                            (malloc=10KB #75)
                            (arena=131KB #3)

-                  Internal (reserved=35999KB +3KB, committed=35999KB +3KB)
                            (malloc=35967KB +3KB #5089 +12)
                            (mmap: reserved=32KB, committed=32KB)

-                    Symbol (reserved=4718KB, committed=4718KB)
                            (malloc=3687KB #30868)
                            (arena=1031KB #1)

-    Native Memory Tracking (reserved=649KB, committed=649KB)
                            (malloc=9KB #100 +1)
                            (tracking overhead=640KB)

-               Arena Chunk (reserved=213KB -1KB, committed=213KB -1KB)
                            (malloc=213KB -1KB)
```
## 针对不同 OS 优化 JVM
### 大页

### 压缩的 oop
