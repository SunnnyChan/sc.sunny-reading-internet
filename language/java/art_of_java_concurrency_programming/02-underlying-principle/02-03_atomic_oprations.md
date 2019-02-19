# 原子操作的实现原理
```md
原子操作是指“不可被中断的一个或一些操作”。
原子操作的实现关乎 处理器 和 JVM。
```
> 首先了解 [CPU 相关术语](../00-refer/CPU-terms.md)

## 处理器如何实现原子操作
```md
处理器会保证基本的内存操作的原子性，处理器保证从内存中读取或写入一个字节是原子的，
即当处理器读取一个字节时，其他处理器不能访问这个字节的内存地址。
最新的处理器能保证对同一个缓存行里进行 16/32/64位 的操作是原子的。

但是复杂的内存操作是不能自动保证其原子性的，比如跨总线宽度、跨多个缓存行或跨页表的访问。
此时，处理器提供总线锁定和缓存锁定来实现复杂内存操作的原子性。
```
### 使用总线锁保证原子性
```md
如果多个处理器同时对共享变量做读写操作（i++就是经典的场景），
共享变量如果被多个处理器操作，这样的操作就不是原子的，可能造成操作的结果与预期不一致。

原因可能是 多个处理器基于从各自的缓存读取变量做操作。

所谓总线锁就是使用 处理器提供的LOCK#信号，当一个处理器在总线上输出此信号时，
其他处理器的请求被阻塞，那么该处理器可以独占内存。
```
### 使用缓存锁保证原子性
```md
总线锁直接把CPU和内存之间的通信锁住了，这会使得其他处理器不能操作内存。
所以总线锁定的开销比较大，实际上只需要保证对某个内存地址的操作是原子的即可，
其他处理器仍可以处理该内存地址之外的数据，目前处理器在某些场合下使用缓存锁定来代替总线锁定进行优化。

频繁使用的内存数据会被缓存，那么原子操作就可以直接在处理器内部缓存中进行，而不需要声明总线锁。
所谓“缓存锁定”指内存区域如果被缓存在处理器的缓存行中，并且在Lock操作期间被锁定，
那么它执行锁操作回写内存时，不在总线上声明LOCK#信号，而是修改内部的内存地址，
并允许他的缓存一致性机制来保证操作的原子性，因为缓存一致性机制会阻止同时修改由两个以上处理器缓存的内存区域数据，
当其他处理器回写已锁定的缓存行时，会使得缓存行无效。
```
* 不使用缓存锁定的情况
```md
1. 当操作的数据不能被缓存在处理器内部时，或操作的数据跨越多个缓存行式
2. 有些处理器不支持缓存锁定。
以上处理器都会调用总线锁定。
```
## Java 如何实现原子操作

### 使用循环 CAS 实现原子操作
```md
JVM 中的 CAS 操作利用 处理器提供的 CMPXCHG 指令实现。
自旋 CAS 的基本思路 是循环进行 CAS 操作，直到成功为止。
```
* 示例 - 基于 CAS 线程安全的计数器 和 非线程安全计数器
```java
public class Counter {

    private AtomicInteger atomicI = new AtomicInteger(0);
    private int           i       = 0;

    public static void main(String[] args) {
        final Counter cas = new Counter();
        List<Thread> ts = new ArrayList<Thread>(600);
        long start = System.currentTimeMillis();
        for (int j = 0; j < 100; j++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < 10000; i++) {
                        cas.count();
                        cas.safeCount();
                    }
                }
            });
            ts.add(t);
        }
        for (Thread t : ts) {
            t.start();

        }
        // 等待所有线程执行完成
        for (Thread t : ts) {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }
        System.out.println(cas.i);
        System.out.println(cas.atomicI.get());
        System.out.println(System.currentTimeMillis() - start);
    }

    /**
     * 使用CAS实现线程安全计数器
     */
    private void safeCount() {
        for (;;) {
            int i = atomicI.get();
            boolean suc = atomicI.compareAndSet(i, ++i);
            if (suc) {
                break;
            }
        }
    }

    /**
     * 非线程安全计数器
     */
    private void count() {
        i++;
    }
}
```
> Java 5 开始，JUC 提供了一些原子操作类（具体参考[Chapter 7]）
### CAS 的问题

* ABA 问题
```md
一个值原来是A，修改为B，又修改为了A，使用CAS进行检查时会发现它没有发生变化。

* 使用版本号
在变量前追加版本号，每次变量更新时把版本号+1。

* AtomicStampedReference
Java 5 开始，Atomic 包提供 AtomicStampedReference 来解决 ABA 问题，
这个类的 CompareAndSet 方法的作用是首先检查当前引用是否等于预期的引用，
并且检查当前标志是否等于预期标志，如果全部相等，则以原子的方式将该引用和标志设置为给定的更新至。
```
* 循环时间长开销大
```md
CAS 如果长时间不成功，会给处理器带来非常大的损耗。
如果JVM能支持处理器提供的 pause 指令，那么效率会有一定的提升。

* pause 指令
有两个作用：
1. 可以延迟流水线执行指令
  使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，而一些处理器上延迟时间是零。
2. 可以避免在退出循环时因内存顺序冲突（Memory Order Violation）
  而引起CPU流水线被清空（CPU pipeline Flush）
```
* 只能保证一个共享变量的原子操作
```md
对于多个共享变量操作时，CAS 无法保证原子操作。这个时候可以使用锁。

还有一个取巧的办法，把多个共享变量合并成一个共享变量来操作。
如：i = 2， j = a，然后用CAS操作 ij=2a

* AtomicReference
Java 5 开始，Atomic 包提供 AtomicReference 类来保证引用对象之间的原子性，
可以把多个变量放在一个对象里来进行 CAS 操作。
```
### 使用锁机制实现原子操作
```md
锁机制保证只有获得锁的线程才能操作锁定的内存区域。

JVM 内部实现了很多种锁，有偏向锁、轻量级锁 和 互斥锁。
除了偏向锁，JVM 实现锁的方式都是用了 CAS 机制，
即当一个线程想进入同步块的时候使用CAS 的方式来获得锁，当它退出同步块时，使用循环CAS释放锁。
```
