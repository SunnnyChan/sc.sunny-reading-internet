# Java 中的 13 个原子操作类
```md
通常 我们会使用 synchronized 保证多线程不会同时更新共享变量。

从Java 5 开始，提供 Atomic 包中的原子操作类提供了一种用法简单、性能高效、
线程安全地更新一个变量的方式。

Atomic 包 提供基本类型、数组、引用、属性4中类型的原子更新方式。
包中的原子操作类基本都是使用 Unsafe 实现的 包装类。
```
## 原子更新基本类型类
```md
AtomicBoolean
AtomicInteger
AtomicLong
```
### AtomicInteger 
* 常用方法
* * int addAndGet(int delta)
```md
以原子方式将输入的数值和实例中的值（AtomicInteger 里的value）相加，并返回结果。
```
* * boolean compareAndSet(int expect, int update)
```md
如果输入的数值等于预期值，则以原子的方式将值更新。
```
* * int getAndIncrement()
```md
以原子方式将值加1，注意这里返回的是自增前的值。
```
* * void lazySet(int newValue)
```md
最终会设置成 newValue，使用 lazySet 设置值后，
可能导致其他线程在之后的一小段时间内还是读取的旧值。
```
* * int getAndSet(int newValue)
```md
以原子方式设置为 newValue，并返回旧值。
```

* 实现
```md
书中使用的是Java 7的实现，Java 8 对 CAS 进行了增强。
```
> 参考[Java 8 中 CAS 的增强](https://www.ktanx.com/blog/p/4693)

* 其他基本类型

## 原子更新数组
```md
AtomicIntegerArray
AtomicLongArray
AtomicReferenceArray
```

### AtomicIntegerArray

## 原子更新引用类型
```md
AtomicReference
AtomicReferenceFieldUpdater
AtomicMarkableReference
```
### AtomicReference


## 原子更新字段类
```md
AtomicIntegerFieldUpdater
AtomicLongFieldUpdater
AtomicStampedReference
```
### AtomicIntegerFieldUpdater