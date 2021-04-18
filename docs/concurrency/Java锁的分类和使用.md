# Java 锁的分类和使用

## Java 锁的种类

Java 锁的分类主要如下：

- 乐观锁/悲观锁
- 独享锁/共享锁
- 互斥锁/读写锁
- 可重入锁
- 公平锁/非公平锁
- 分段锁
- 偏向锁/轻量级锁/重量级锁
- 自旋锁

以上是一些锁的名词，这些分类并不是全是指锁的状态，有的指锁的特性，有的指锁的设计。

## 乐观锁 / 悲观锁

乐观锁与悲观锁并不是特指某两种类型的锁，是人们定义出来的概念或思想，主要是指看待并发同步的角度。

- **乐观锁**

顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。乐观锁适用于多读的应用类型，这样可以提高吞吐量，在 Java 中`java.util.concurrent.atomic`包下面的原子变量类就是使用了乐观锁的一种实现方式 `CAS`(`Compare and Swap`比较并交换)实现的。

乐观锁总是认为不存在并发问题，每次去取数据的时候，总认为不会有其他线程对数据进行修改，因此不会上锁。但是在更新时会判断其他线程在这之前有没有对数据进行修改，一般会使用“数据版本机制”或“CAS操作”来实现。

1）数据版本机制

实现数据版本一般有两种，第一种是使用版本号，第二种是使用时间戳。以版本号方式为例。

版本号方式：一般是在数据表中加上一个数据版本号`version`字段，表示数据被修改的次数，当数据被修改时，`version`值会加1。当线程A要更新数据值时，在读取数据的同时也会读取`version`值，在提交更新时，若刚才读取到的`version`值为当前数据库中的`version`值相等时才更新，否则重试更新操作，直到更新成功。比如如下SQL代码：

```sql
update table set xxx=#{xxx}, version=version+1 where id=#{id} and version=#{version};
```

2）CAS操作

CAS（`Compare and Swap`比较并交换），当多个线程尝试使用`CAS`同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。

CAS 操作中包含三个操作数——需要读写的内存位置(`V`)、进行比较的预期原值(`A`)和拟写入的新值(`B`)。如果内存位置V的值与预期原值A相匹配，那么处理器会自动将该位置值更新为新值`B`，否则处理器不做任何操作。

- **悲观锁**

悲观锁总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞直到它拿到锁。比如Java 里面的同步原语`synchronized`关键字的实现就是悲观锁。

悲观锁适合写操作非常多的场景，乐观锁适合读操作非常多的场景，不加锁会带来大量的性能提升。

悲观锁在 Java 中的使用，就是利用各种锁。乐观锁在 Java 中的使用，是无锁编程，常常采用的是`CAS`算法，典型的例子就是原子类，通过`CAS`自旋实现原子操作的更新。悲观锁认为对于同一个数据的并发操作，一定会发生修改的，哪怕没有修改，也会认为修改。因此对于同一份数据的并发操作，悲观锁采取加锁的形式。悲观的认为，不加锁并发操作一定会出问题。

在对任意记录进行修改前，先尝试为该记录加上排他锁（`exclusive locking`）。如果加锁失败，说明该记录正在被修改，那么当前查询可能要等待或者抛出异常。具体响应方式由开发者根据实际需要决定。如果成功加锁，那么就可以对记录做修改，事务完成后就会解锁了。期间如果有其他对该记录做修改或加排他锁的操作，都会等待我们解锁或直接抛出异常。

## 独享锁 / 共享锁

独享锁是指该锁一次只能被一个线程所持有。

共享锁是指该锁可被多个线程所持有。

对于`Java ReentrantLock`而言，其是独享锁。但是对于`Lock`的另一个实现类`ReadWriteLock`，其读锁是共享锁，其写锁是独享锁。

读锁的共享锁可保证并发读是非常高效的，读写，写读，写写的过程是互斥的。

独享锁与共享锁也是通过`AQS`来实现的，通过实现不同的方法，来实现独享或者共享。

对于`synchronized`而言，当然是独享锁。

## 互斥锁 / 读写锁

上面讲的独享锁/共享锁就是一种广义的说法，互斥锁/读写锁就是具体的实现。

互斥锁在`Java`中的具体实现就是`ReentrantLock`。

读写锁在`Java`中的具体实现就是`ReadWriteLock`。

## 可重入锁

可重入锁又名递归锁，是指在同一个线程在外层方法获取锁的时候，在进入内层方法会自动获取锁。说的有点抽象，下面会有一个代码的示例。

对于`Java ReetrantLock`而言，从名字就可以看出是一个重入锁，其名字是`Re entrant Lock`重新进入锁。

对于`synchronized`而言，也是一个可重入锁。可重入锁的一个好处是可一定程度避免死锁。

```java
synchronized void setA() throws Exception{
　　Thread.sleep(1000);
　　setB();
}

synchronized void setB() throws Exception{
　　Thread.sleep(1000);
}
```

上面的代码就是一个可重入锁的一个特点。如果不是可重入锁的话，setB 可能不会被当前线程执行，可能造成死锁。

## 公平锁 / 非公平锁

公平锁是指多个线程按照申请锁的顺序来获取锁。

非公平锁是指多个线程获取锁的顺序并不是按照申请锁的顺序，有可能后申请的线程比先申请的线程优先获取锁。有可能，会造成优先级反转或者饥饿现象。

对于`Java ReetrantLock`而言，通过构造函数指定该锁是否是公平锁，默认是非公平锁。非公平锁的优点在于吞吐量比公平锁大。

对于`synchronized`而言，也是一种非公平锁。由于其并不像`ReentrantLock`是通过`AQS`的来实现线程调度，所以并没有任何办法使其变成公平锁。

## 分段锁

分段锁其实是一种锁的设计，并不是具体的一种锁，对于`ConcurrentHashMap`而言，其并发的实现就是通过分段锁的形式来实现高效的并发操作。

我们以`ConcurrentHashMap`来说一下分段锁的含义以及设计思想，`ConcurrentHashMap`中的分段锁称为`Segment`，它即类似于`HashMap`（`JDK7`和`JDK8`中`HashMap`的实现）的结构，即内部拥有一个`Entry`数组，数组中的每个元素又是一个链表；同时又是一个`ReentrantLock`（`Segment`继承了`ReentrantLock`）。

当需要`put`元素的时候，并不是对整个`hashmap`进行加锁，而是先通过`hashcode`来知道他要放在哪一个分段中，然后对这个分段进行加锁，所以当多线程`put`的时候，只要不是放在一个分段中，就实现了真正的并行的插入。

但是，在统计`size`的时候，可就是获取`hashmap`全局信息的时候，就需要获取所有的分段锁才能统计。

分段锁的设计目的是细化锁的粒度，当操作不需要更新整个数组的时候，就仅仅针对数组中的一项进行加锁操作。

## 偏向锁 / 轻量级锁 / 重量级锁

这三种锁是指锁的状态，并且是针对`synchronized`。在 Java 5 通过引入锁升级的机制来实现高效`synchronized`。这三种锁的状态是通过对象监视器在对象头中的字段来表明的。

偏向锁是指一段同步代码一直被一个线程所访问，那么该线程会自动获取锁。降低获取锁的代价。

轻量级锁是指当锁是偏向锁的时候，被另一个线程所访问，偏向锁就会升级为轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，提高性能。

重量级锁是指当锁为轻量级锁的时候，另一个线程虽然是自旋，但自旋不会一直持续下去，当自旋一定次数的时候，还没有获取到锁，就会进入阻塞，该锁膨胀为重量级锁。重量级锁会让他申请的线程进入阻塞，性能降低。

## 自旋锁

在 Java 中，自旋锁是指尝试获取锁的线程不会立即阻塞，而是采用循环的方式去尝试获取锁，这样的好处是减少线程上下文切换的消耗，缺点是循环会消耗`CPU`。

## 锁的基础类

- **AQS**

`AbstractQueuedSynchronized`抽象队列式的同步器，`AQS`定义了一套多线程访问共享资源的同步器框架，许多同步类实现都依赖于它，如常用的ReentrantLock / Semaphore / CountDownLatch…

![721070-20170504110246211-10684485.png](https://i.loli.net/2021/03/01/qTZXYFGdcmjk37u.png)

`AQS`维护了一个`volatile int state`( 代表共享资源 ) 和一个`FIFO`线程等待队列（多线程争用资源被阻塞时会进入此队列）。

state 的访问方式有三种：

```java
1 getState()
2 setState()
3 compareAndSetState
```

`AQS`定义两种资源共享方式：`Exclusive`（独占，只有一个线程能执行，如`ReentrantLock`）和`Share`（共享，多个线程可同时执行，如`Semaphore/CountDownLatch`）。

不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源`state`的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），`AQS`已经在顶层实现好了。自定义同步器实现时主要实现以下几种方法：

```java
1 isHeldExclusively(): 该线程是否正在独占资源。只有用到condition才需要去实现它。
2 tryAquire(int): 独占方式。尝试获取资源，成功则返回true，失败则返回false。
3 tryRelease(int): 独占方式。尝试释放资源，成功则返回true，失败则返回false。
4 tryAcquireShared(int): 共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
5 tryReleaseShared(int): 共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。
```

以`ReentrantLock`为例，`state`初始化为0，表示未锁定状态。A 线程`lock()` 时，会调用`tryAcquire()`独占该锁并将`state+1`。此后，其他线程再`tryAcquire()`时就会失败，直到A线程`unlock()`到`state = 0`（即释放锁）为止，其他线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（`state`会累加），这就是可重入的概念。但要注意，获取多少次就要释放多少次，这样才能保证`state`是能回到零态的。

再以`CountDownLatch`为例，任务分为 N 个子线程去执行，`state`为初始化为 N（注意 N 要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后`countDown()`一次，`state`会 CAS 减1。等到所有子线程都执行完后（即`state = 0`），会`unpark()`主调用线程，然后主调用线程就会`await()`函数返回，继续后余动作。

一般来说，自定义同步器要么是独占方式，要么是共享方式，它们也只需实现`tryAcquire-tryRelease`、`tryAcquireShared-tryReleaseShared`中的一种即可。但`AQS`也支持自定义同步器同时实现独占和共享两种方式，如`ReentrantReadWriteLock`。

- **CAS**

CAS（`Compare and Swap`比较并交换）是乐观锁技术，当多个线程尝试使用 CAS 同时更新同一个变量时，只有其中一个线程能更新变量的值，而其他线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。

CAS 操作中包含三个操作数——需要读写的内存位置（V）、进行比较的预期原值（A）和拟写入的新值（B）。如果内存位置 V 的值与预期原值A相匹配，那么处理器会自动将该位置值更新为新值 B，否则处理器不做任何操作。无论哪种情况，它都会在 CAS 指令之前返回该位置的值（在 CAS 的一些特殊情况下将仅返回 CAS 是否成功，而不提取当前值）。CAS 有效地说明了“我认为位置 V 应该包含值 A；如果包含该值，则将B放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可”。这其实和乐观锁的冲突检查+数据更新的原理是一样的。

JAVA 对 CAS 的支持：

在 JDK1.5 中新增`java.util.concurrent`包就是建立在 CAS 之上的。相对于`synchronized`这种阻塞算法，CAS 是非阻塞算法的一种常见实现。所以`java.util.concurrent`包中的`AtomicInteger`为例，看一下在不使用锁的情况下是如何保证线程安全的。主要理解`getAndIncrement`方法，该方法的作用相当于`++i`操作。详见 [Atomic原子类](concurrency/Atomic原子类.md)。

```java
public class AtomicInteger extends Number implements java.io.Serializable{
　　private volatile int value;
　　public final int get(){
　　　　return value;
　　}

　　 public final int getAndIncrement(){
　　　　for (;;){
　　　　　　int current = get();
　　　　　　int next = current + 1;
　　　　　　if (compareAndSet(current, next))
　　　　　　return current;
　　　　}
　　}

　　public final boolean compareAndSet(int expect, int update){
　　　　return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
　　}
}
```

## 锁的使用

- **synchronized**

synchronized 可重入锁验证，详见 [synchronized 关键字原理](concurrency/synchronized关键字原理.md)。

```java
public class SynchronizedDemo {

    private static Object obj = new Object();

    public static void main(String[] args) {
        Runnable sellTicket = new Runnable() {
            @Override
            public void run() {
                synchronized (SynchronizedDemo.class) {
                    log.info("i am run");
                    test01(); }
            }
            public void test01() {
                synchronized (SynchronizedDemo.class) {
                    log.info("i am test01"); }
            } };
        new Thread(sellTicket).start();
    }

}
```

运行结果

```bash
i am run
i am test01
```

- **ReentrantLock**

ReentrantLock 既可以构造公平锁又可以构造非公平锁，默认为非公平锁。

```java
public class LockDemo {

    private static Lock lock = new ReentrantLock();
    private static CountDownLatch countDownLatch = new CountDownLatch (10);
    private static int num = 0;

    public static void inCreate() {
        try{
           // 获取锁
           lock.lock ();
           num++; 
        }
		finally{
           // 释放锁
           lock.unlock ();            
        }
    }

    public static void main(String[] args) {

        try{
            for (int i = 0; i < 10; i++) {
                new Thread (() -> {
                    for (int j = 0; j < 10000; j++) {
                        inCreate ();
                    }
                    // 线程执行结束执行countDown，对计数减1
                    countDownLatch.countDown ();
                }).start ();
            }

            countDownLatch.await();
            log.info(num);
        }
        catch(InterruptedException e){
            e.printStackTrace();
        }
    }
}
```

运行结果

```bash
100000
```

`ReentrantLock`是可重入的独占锁。比起`synchronized`功能更加丰富，支持公平锁实现，支持中断响应以及限时等待等等。可以配合一个或多个`Condition`条件方便的实现等待通知机制。

`ReentrantLock`类中带有两个构造函数，一个是默认的构造函数，不带任何参数；一个是带有`fair`参数的构造函数。

```java
public ReentrantLock() {
  sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
  sync = fair ? new FairSync() : new NonfairSync();
}
```

第二个构造函数也是判断`ReentrantLock`是否是公平锁的条件，如果`fair`为`true`，则会创建一个公平锁的实现，也就是`new FairSync()`，如果`fair`为`false`，则会创建一个 非公平锁的实现，也就是`new NonfairSync()`，默认的情况下创建的是非公平锁修改上面例程的代码构造方法为：

```java
ReentrantLock reentrantLock = new ReentrantLock(true);
```

对于`ReentrantLock`公平锁和非公平锁的原理和用法，详见 [ReentrantLock 原理](concurrency/ReentrantLock原理.md)。

- **ReentrantReadWriteLock**

读写锁的性能都会比排他锁要好，因为大多数场景读是多于写的。在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量。Java 并发包提供读写锁的实现是`ReentrantReadWriteLock`。

| 特性       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| 公平性选择 | 支持非公平(默认)和公平的锁获取方式，吞吐量还是非公平优于公平 |
| 重进入     | 该锁支持重进入，以读写线程为例：读线程在获取了读锁之后，能够再次获取读锁。而写线程在获取了写锁之后能够再次获取写锁，同时也可以获取读锁 |
| 锁降级     | 遵循获取写锁、获取读锁再释放写锁的次序，写锁能够降级成为读锁 |

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class MyLockTest {

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    Cache.put("key", new String(Thread.currentThread().getName() + " joke"));
                }
            }, "threadW-" + i).start();
            new Thread(new Runnable() {
                @Override
                public void run() {
                    System.out.println(Cache.get("key"));
                }
            }, "threadR-" + i).start();
            new Thread(new Runnable() {
                @Override
                public void run() {
                    Cache.clear();
                }
            }, "threadC-" + i).start();
        }
    }
}

class Cache {
    static Map<String, Object> map = new HashMap<String, Object>();
    static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    static Lock r = rwl.readLock();
    static Lock w = rwl.writeLock();

    // 获取一个key对应的value
    public static final Object get(String key) {
        r.lock();
        try {
            log.info("get " + Thread.currentThread().getName());
            return map.get(key);
        } finally {
            r.unlock();
        }
    }

    // 设置key对应的value，并返回旧有的value
    public static final Object put(String key, Object value) {
        w.lock();
        try {
            log.info("put " + Thread.currentThread().getName());
            return map.put(key, value);
        } finally {
            w.unlock();
        }
    }

    // 清空所有的内容
    public static final void clear() {
        w.lock();
        try {
            log.info("clear " + Thread.currentThread().getName());
            map.clear();
        } finally {
            w.unlock();
        }
    }
}
```

运行结果

```ASN.1
put threadW-0
clear threadC-1
put threadW-1
get threadR-1
threadW-1 joke
put threadW-2
get threadR-0
threadW-2 joke
clear threadC-0
get threadR-2
null
clear threadC-4
clear threadC-2
clear threadC-3
put threadW-4
put threadW-3
get threadR-3
threadW-3 joke
put threadW-5
get threadR-4
threadW-5 joke
clear threadC-5
put threadW-6
put threadW-7
get threadR-7
threadW-7 joke
get threadR-5
threadW-7 joke
get threadR-6
threadW-7 joke
clear threadC-6
clear threadC-7
put threadW-8
clear threadC-8
put threadW-9
get threadR-9
threadW-9 joke
clear threadC-9
get threadR-8
null
```

可看到普通 HashMap 在多线程中数据可见性正常。

