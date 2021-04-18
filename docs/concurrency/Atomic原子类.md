# Atomic 原子类

## 线程不安全

当多个线程访问统一资源时，如果没有做线程同步，可能造成线程不安全，导致数据出错。如下代码：

```java
@Slf4j
public class ThreadUnsafe {

    // 用于计数的统计变量
    private static int count = 0;
    
    // 线程数量
    private static final int Thread_Count = 10;
    
    // 线程池
    private static ExecutorService executorService = Executors.newCachedThreadPool();
    
    // 初始化值和线程数一致
    private static CountDownLatch downLatch = new CountDownLatch(Thread_Count);

    // 测试
    public static void main(String[] args) throws Exception{
        for (int i = 0; i < Thread_Count; i++) {
            executorService.execute(() -> {
                for (int j = 0; j < 1000; j++) {  // 每个线程执行1000次++操作
                    count++;
                }
                // 一个线程执行完
                downLatch.countDown();
            });
        }
        // 等待所有线程执行完
        downLatch.await();
        log.info("count is {}", count);
    }
}
```

当多个线程对`count`变量计数，每个线程加1000次，10个线程理想状态下是加10000次，但实际情况并非如此。

上面的代码执行5次后，打印出来的`count`值分别为 7130，8290，9370，8790，8132。从测试的结果看出，`count`的值并不是我们认为的10000次，而都是小于10000次。

之所以出现上面的结果就是因为`count++`操作并非原子的。它其实被分成了三步：

```java
tp1 = count;  //1
tp2 = tp1 + 1;  //2
count = tp2;  //3
```

所以 ，如果有两个线程`m`和`n`要执行`count++`操作。如果是理想状态下，`m`和`n`线程依次执行，`m`先执行完后，`n`再执行，即`m1 -> m2 -> m3 -> n1 -> n2 -> n3`，那么结果是没问题的。但是如果线程代码的执行顺序是`m1 -> n1 -> m2 -> n2 -> m3 -> n3`，那么很明显结果就会出错，主要原因就是 [volatile 关键字](concurrency/volatile关键字.md) 文中提到的 java 内存模型关系。

而上面的测试结果也正是由于没有做线程同步，导致的线程在执行`count++`时，乱序执行后`count`的数值就不对了。

## 原子操作

- **使用 synchronized 实现线程同步**

对上面的代码做一些改造，对`count++`操作加入`synchronized`关键字修饰，实现线程同步，以保证每个线程在执行`count++`时，必须执行完成后，另一个线程才开始执行的。代码如下：

```java
@Slf4j
public class ThreadUnsafe {

    // 用于计数的统计变量
    private static int count = 0;
    // 线程数量
    private static final int Thread_Count = 10;
    // 线程池
    private static ExecutorService executorService = Executors.newCachedThreadPool();
    // 初始化值和线程数一致
    private static CountDownLatch downLatch = new CountDownLatch(Thread_Count);

    public static void main(String[] args) throws Exception{
        for (int i = 0; i < Thread_Count; i++) {
            executorService.execute(() -> {
                for (int j = 0; j < 1000; j++) {  // 每个线程执行1000次++操作
                    synchronized (ThreadUnsafe.class) {
                        count++;
                    }
                }
                // 一个线程执行完
                downLatch.countDown();
            });
        }
        // 等待所有线程执行完
        downLatch.await();
        log.info("count is {}", count);
    }
}
```

将线程不安全的测试代码添加`synchronized`关键字进行线程同步，保证线程在执行`count++`操作时，是依次执行完后，后面的线程才开始执行的。`synchronized`关键字可以实现原子性和可见性。

将上面的代码执行5次后，打印出来的`count`值均为10000，已经是正确结果了。

## 原子类

在`JDK1.5`中新增了`java.util.concurrent(J.U.C)`包，它建立在`CAS`之上。而`CAS`采用了乐观锁思路，是非阻塞算法的一种常实现，相对于`synchronized`这种阻塞算法，它的性能更好。

- **乐观锁与 CAS**

在`JDK5`之前，Java 是靠`synchronized`关键字保证线程同步的，这会导致有锁，锁机制存在以下问题：

在多线程竞争下，加锁和释放锁会导致比较多的上下文切换和调度延时，引起性能问题；

一个线程持有锁后，会导致其他所有等待该锁的线程挂起；

如果一个优先级高的线程等待一个优先级低的线程释放锁会导致线程优先级倒置，引起风险；

独占锁采用的是悲观锁思路。`synchronized`就是一种独占锁，它会导致其他所有需要锁的线程挂起。而另一种更加有效的锁就是乐观锁，`CAS`就是一种乐观锁。乐观锁，严格来说并不是锁。它是通过原子性来保证数据的同步，比如说数据库的乐观锁，通过版本控制`mvcc`来实现，所以`CAS`不会保证线程同步，只是乐观地认为在数据更新期间没有其他线程参与。

`CAS`是一种无锁算法。无锁编程，即在不使用锁的情况下实现多线程间的同步，也就是在没有线程被阻塞挂起的情况下实现变量的同步。

`CAS`算法即是：`Compare And Swap`,比较并替换。

`CAS`算法存在着三个参数，内存值`V`，期望值`A`，以及需要更新的值`B`。当且仅当内存值`V`和期望值`A`相等的时候，才会将内存值修改为`B`，否则什么也不做，继续循环检查;

由于`CAS`是`CPU`指令，我们只能通过`JNI`与操作系统交互，关于`CAS`的方法都在`sun.misc`包下`Unsafe`的类里，`java.util.concurrent.atomic`包下的原子类等通过`CAS`来实现原子操作。

`CAS`特点：

`CAS`是原子操作，保证并发安全，而不能保证并发同步

`CAS`是 CPU 的一个指令（需要 JNI 调用 Native 方法，才能调用 CPU 的指令）

`CAS`是非阻塞的、轻量级的乐观锁

- **AtomicInteger 实现**

`JDK`提供了原子操作类，指的是`java.util.concurrent.atomic`包下，一系列以`Atomic`开头的包装类。例如`AtomicBoolean`，`AtomicInteger`，`AtomicLong`。它们分别用于`Boolean`，`Integer`，`Long`类型的原子性操作。以下是`AtomicInteger`部分源代码：

```java
    static {
        try {
            //获取value属性值在内存中的地址偏移量
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

    public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }
```

`AtomicInteger`的`getAndIncrement`调用了`Unsafe`的`getAndInt`方法完成 +1 原子操作。`Unsafe`类的`getAndInt`方法源码如下：

```java
    //var1是this指针，var2是地址偏移量，var4是自增值，是自增1还是自增N
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            //获取内存值
            var5 = this.getIntVolatile(var1, var2);

            //var5是期望值，var5 + var4是要更新的值
            //这个操作就是调用CAS的JNI,每个线程将自己内存里的内存值与var5期望值E作比较，如果相同，就将内存值更新为var5 + var4，否则做自旋操作
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
        
        return var5;
    }
```

实现原子操作是基于`compareAndSwapInt`方法，更新前先取出内存值进行比较，和期望值一致后才更新。 

