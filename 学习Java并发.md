## Java 并发

### AtomicIntegerFieldUpdater

通过反射方式去保证volatile int 的增减操作是原子的。

好处：
另一种方式就是将int 改为 AtomicInteger类型，那么所有使用这个变量的地方都要修改成AtomicInteger 对应的方法。改动很大，同时违背了开闭原则。

使用AtomicIntegerFieldUpdater后，不需要大量修改所有使用int 变量的操作地方，只修改保证原子性的地方。从另一个方面看，在不需要原子性的地方，就是原来的int 操作，减少了损耗。这样想原子就原子，不想原子也可以不原子，更细粒度化了。

参考：https://www.jianshu.com/p/b83d67a742e8

使用场景：
可以使用volatile + AtomicIntegerFieldUpdater 替代 AtomicInterger，适合读取较多的场景。


### AtomicReferenceFieldUpdater

通过反射方式去操作对象的volatile变量。

使用场景：
读取和设置操作，读取get，利用volatile的可读。然后设置新值是原子操作。


### VarHandler

Java 9 除了模块化支持，还有新的API - VarHandler 对内存操作 （替代直接使用Unsafe class）。

如果要原子性地增加某个字段的值，目前有下面三种方式：

- 使用AtomicInteger来达到这种效果，这种间接管理方式增加了空间开销，还会导致额外的并发问题；
- 使用原子性的FieldUpdaters，利用了反射机制，操作开销也会更大；
- 使用sun.misc.Unsafe提供的JVM内置函数API，虽然这种方式比较快，但它会损害安全性和可移植性。

在 VarHandle 出现之前，这些潜在的问题会随着原子API的不断扩大而越来越遭。VarHandle 的出现替代了 java.util.concurrent.atomic 和 sun.misc.Unsafe 的部分操作。并且提供了一系列标准的内存屏障操作，用于更加细粒度的控制内存排序


参考： https://blog.csdn.net/sench_z/article/details/79793741

### 内存屏障

volatile StoreLoad Barrier


### 锁

自旋锁是乐观锁的一种。自旋是同步资源获取失败后，继续循环尝试获取资源，避免CPU 切换挂起的时间消耗。而乐观锁只是指不加锁，对同步资源进行更新，如果更新失败，可能继续循环（自旋锁），也可以报错。