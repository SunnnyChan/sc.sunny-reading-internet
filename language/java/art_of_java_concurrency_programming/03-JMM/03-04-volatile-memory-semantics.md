# volatile 的内存语义
```md
当声明变量为 volatile 后，对这个变量的读写将会很特别。
```
## volatile 的特性
* 可见性
* 原子性
```md
对任意的那个 volatile 变量的读写具有原子性，但类似 volatile++ 这种复合操作不具有原子性。
```
## volatile 写读建立的 happens-before 关系

## volatile 写读 的内存语义
```md
当写一个volatile 变量时，JMM 会本地内存中的共享变量值刷新至主内存。
```
## volatile 内存语义的实现

## JSR-133 为什么要增强 volatile 的内存语义