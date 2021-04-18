# CompareAndSet (CAS)

## CAS 简介

`CAS：Compare and Swap`, 翻译成比较并交换。`java.util.concurrent（JUC`）包中借助`CAS`实现了区别于`synchronized`同步锁的一种乐观锁，使用这些类在多核CPU 的机器上会有比较好的性能。

`CAS`有3个操作数，内存值`V`，旧的预期值`A`，要修改的新值`B`。当且仅当预期值`A`和内存值`V`相同时，将内存值`V`修改为`B`，否则什么都不做。

我们针对`AtomicInteger`的`incrementAndGet`做深入分析。`AtomicInteger`类`compareAndSet`通过原子操作实现了`CAS`操作，最底层基于汇编语言实现。简单说一下原子操作的概念，“原子”代表最小的单位，所以原子操作可以看做最小的执行单位，该操作在执行完毕前不会被任何其他任务或事件打断。

`CAS`是`Compare And Set`的一个简称，如下理解：

- 已知当前内存里面的值`current`和预期要修改成的新值`new`传入

- 内存中`AtomicInteger`对象地址对应的真实值 (因为有可能被修改)`real`与`current`对比相等表示`real`未被修改过，是 “安全” 的，将`new`赋给`real`结束然后返回；不相等说明`real`已经被修改，结束并重新执行上一步直到修改成功。

`CAS`相比`synchronized`，避免了锁的使用，总体性能比`synchronized`高很多。

## 代码实例

`compareAndSet`典型应用为计数，如`i++`，`++i`，这里以`i++`为例：

```java
/**
 * Atomically increments by one the current value.
 *
 * @return the updated value
 */
public final int incrementAndGet() {
    for (;;) {
        //获取当前值
        int current = get();
        //设置期望值
        int next = current + 1;
        //调用Native方法compareAndSet，执行CAS操作
        if (compareAndSet(current, next))
            //成功后才会返回期望值，否则无线循环
            return next;
    }
}
```

`compareAndSet`方法实现：

`JDK`文档对该方法的说明如下：如果当前状态值等于预期值，则以原子方式将同步状态设置为给定的更新值。

```java
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```

这里解释一下`valueOffset`变量，首先`valueOffset`的初始化在`static`静态代码块里面，代表相对起始内存地址的字节相对偏移量。

```java
private static final long valueOffset;
    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }    

    private volatile int value;
 
    /**
     * Creates a new AtomicInteger with the given initial value.
     *
     * @param initialValue the initial value
     */
    public AtomicInteger(int initialValue) {
        value = initialValue;
    }
 
    /**
     * Creates a new AtomicInteger with initial value {@code 0}.
     */
    public AtomicInteger() {
    }
```

在生成一个`AtomicInteger`对象后，可以看做生成了一段内存，对象中各个字段按一定顺序放在这段内存中，字段可能不是连续放置的，`unsafe.objectFieldOffset (Field f)`这个方法返回了 "`value`" 属性字段相对于`AtomicInteger`对象的起始内存地址的字节相对偏移量。`value`是一个`volatile`变量，不同线程对这个变量进行操作时具有可见性，修改与写入操作都会立即写入主存中，并通知其他`cpu`中该变量缓存无效，保证了每次读取的都是最新的值。

## 底层原理

`CAS`通过调用`JNI`的代码实现的。`JNI : Java Native Interface`为`JAVA`本地调用，允许`java`调用其他语言。而`compareAndSwapInt`就是借助`C`来调用`CPU`底层指令实现的。

下面从分析比较常用的`CPU（intel x86）`来解释`CAS`的实现原理。`sun.misc.Unsafe`类的`compareAndSwapInt()`方法的源代码：

```java
public final native boolean compareAndSwapInt(Object o, long offset,
                                              int expected,
                                              int x);
 

//可以看到这是个本地方法调用。这个本地方法在openjdk中依次调用的c++代码为：unsafe.cpp，atomic.cpp和atomicwindowsx86.inline.hpp。//这个本地方法的最终实现在openjdk的如下位置：openjdk-7-fcs-src-b147-27jun2011\openjdk\hotspot\src\oscpu\windowsx86\vm\ atomicwindowsx86.inline.hpp（对应于windows操作系统，X86处理器）。

下面是对应于intel x86处理器的源代码的片段：
// Adding a lock prefix to an instruction on MP machine
// VC++ doesn't like the lock prefix to be on a single line
// so we can't insert a label after the lock prefix.
// By emitting a lock prefix, we can define a label after it.
#define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                       __asm je L0      \
                       __asm _emit 0xF0 \
                       __asm L0:

inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  // alternative for InterlockedCompareExchange
  int mp = os::is_MP();
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    LOCK_IF_MP(mp)
    cmpxchg dword ptr [edx], ecx
  }
}
```

如上面源代码所示，程序会根据当前处理器的类型来决定是否为`cmpxchg`指令添加`lock`前缀。如果程序是在多处理器上运行，就为`cmpxchg`指令加上`lock`前缀（`lock cmpxchg`）。反之，如果程序是在单处理器上运行，就省略 lock 前缀（单处理器自身会维护单处理器内的顺序一致性，不需要`lock`前缀提供的内存屏障效果）。

`intel`的手册对lock前缀的说明如下：

- 确保对内存的读-改-写操作原子执行。在`Pentium`及`Pentium`之前的处理器中，带有`lock`前缀的指令在执行期间会锁住总线，使得其他处理器暂时无法通过总线访问内存。很显然，这会带来昂贵的开销。从`Pentium 4，Intel Xeon`及`P6`处理器开始，`intel`在原有总线锁的基础上做了一个很有意义的优化：如果要访问的内存区域（`area of memory`）在lock前缀指令执行期间已经在处理器内部的缓存中被锁定（即包含该内存区域的缓存行当前处于独占或以修改状态），并且该内存区域被完全包含在单个缓存行（`cache line`）中，那么处理器将直接执行该指令。由于在指令执行期间该缓存行会一直被锁定，其它处理器无法读/写该指令要访问的内存区域，因此能保证指令执行的原子性。这个操作过程叫做缓存锁定（`cache locking`），缓存锁定将大大降低`lock`前缀指令的执行开销，但是当多处理器之间的竞争程度很高或者指令访问的内存地址未对齐时，仍然会锁住总线。

- 禁止该指令与之前和之后的读和写指令重排序。

- 把写缓冲区中的所有数据刷新到内存中。

总的来说，`Atomic`实现了高效无锁（底层还是用到排它锁，不过底层处理比`java`层处理要快很多) 与线程安全 (`volatile`变量特性)，`CAS`一般适用于计数；多线程编程也适用，多个线程执行`AtomicXXX`类下面的方法，当某个线程执行的时候具有排他性，在执行方法中不会被打断，直至当前线程完成才会执行其他的线程。

## 相关问题

`CAS`虽然很高效的解决原子操作，但是`CAS`仍然存在如下三大问题：

- **ABA 问题**。因为`CAS`需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是`A`，变成了`B`，又变成了`A`，那么使用`CAS`进行检查时会发现它的值没有发生变化，但是实际上却变化了。`ABA`问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加1，那么 `A－B－A`就会变成`1A - 2B－3A`，而`1A`和`3A`肯定是不一样的。从`Java1.5`开始`JDK`的`atomic`包里提供了一个类`AtomicStampedReference`来解决`ABA`问题。这个类的`compareAndSet`方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。关于`ABA`问题参考文档: http://blog.hesey.net/2011/09/resolve-aba-by-atomicstampedreference.html

- **循环时间长开销大**。自旋`CAS`如果长时间不成功，会给`CPU`带来非常大的执行开销。

- **只能保证一个共享变量的原子操作**。当对一个共享变量执行操作时，我们可以使用循环`CAS`的方式来保证原子操作，但是对多个共享变量操作时，循环`CAS`就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量`i＝2, j = a`，合并一下`ij = 2a`，然后用`CAS`来操作`ij`。从`Java1.5`开始`JDK`提供了`AtomicReference`类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行`CAS`操作。