# synchronized 关键字原理

在Java中，`synchronized`是我们罪常使用的锁，但在 JDK1.5之前`synchronized`是一个重量级锁，相对于 juc 包中的`Lock`，显得比较笨重。在 Java 6 之后 Java 官⽅从 JVM 层⾯对`synchronized`进行了大量优化，使得`synchronized`锁效率提高了很多。

## 锁使用

- #### synchronized 作用


1）原子性：所谓原子性就是指一个操作或者多个操作，要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。被`synchronized`修饰的类或对象的所有操作都是原子的，因为在执行操作之前必须先获得类或对象的锁，直到执行完才能释放。

2）可见性：可见性是指多个线程访问一个资源时，该资源的状态、值信息等对于其他线程都是可见的。`synchronized`和`volatile`都具有可见性，其中`synchronized`对一个类或对象加锁时，一个线程如果要访问该类或对象必须先获得它的锁，而这个锁的状态对于其他任何线程都是可见的，并且在释放锁之前会将对变量的修改刷新到共享内存当中，保证资源变量的可见性。

3）有序性：有序性是指程序执行的顺序按照代码先后执行。`synchronized`和`volatile`都具有有序性，Java 允许编译器和处理器对指令进行重排，但是指令重排并不会影响单线程的顺序，它影响的是多线程并发执行的顺序性。`synchronized`保证了每个时刻都只有一个线程访问同步代码块，也就确定了线程执行同步代码块是分先后顺序的，保证了有序性。

- #### synchronized 使用方法


1）修饰实例方法：对当前对象实例加锁，进入同步方法前要获得当前对象实例的锁。

```java
synchronized void method() {
  //业务代码
}
```

2）修饰静态方法：也就是对当前类加锁，会作用于类的所有对象实例 ，进入同步代码前要获得 当前`class`的锁。因为静态成员不属于任何一个实例对象，是类成员，近属于类。类对象（`class`）的锁和`new`出来的实例对象的锁是不同的。当一个线程 A 调用一个实例对象的非静态`synchronized`方法，而线程 B 调用这个实例对象所属类的静态`synchronized`方法时，这种情况是允许的，不会发生互斥现象，因为访问静态`synchronized`方法占用的锁是当前类的锁，而访问非静态`synchronized`方法占用的锁是`new`出来的实例对象锁，二者是不同的。

```java
synchronized void staic method() {
  //业务代码
}
```

3）修饰代码块 ：指定加锁对象，对给定对象/类加锁。`synchronized(object)`表示进入同步代码库前要获得指定对象`object`的锁。`synchronized(类.class)`表示进入同步代码前要获得指定类`class`对象的锁

```java
synchronized(this) {
  //业务代码
}
```

- #### **总结**


`synchronized`关键字加到`static`静态方法和`synchronized(class)`代码块上都是是给 class 类加锁。`synchronized`关键字加到实例方法上是给对象实例加锁。如下是`synchronized`的使用实例：

```java
public class Singleton {
    //保证有序性，防止指令重排
    private volatile static Singleton uniqueInstance;

    private Singleton() {
    }

    public  static Singleton getUniqueInstance() {
       //先判断对象是否已经实例过，没有实例化过才进入加锁代码
        if (uniqueInstance == null) {
            //类对象加锁
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
```

## 锁特性

- #### 可重入性


`synchronized`是可重入锁。当一个线程再次请求自己已经持有的对象锁的临界资源时，这种情况属于重入锁。在 java 中`synchronized`是基于原子性的内部锁机制的，是可重入的，因此在一个线程调用`synchronized`方法的同时在其方法体内部调用该对象的另一个`synchronized`方法，也就是说一个线程得到一个对象锁后再次请求该对象锁时，是可以的。这就是`synchronized`的可重入性。

- #### 不可中断


1） 不可中断的意思是等待获取锁的时候不可中断，拿到锁之后可中断，没获取到锁的情况下，中断操作一直不会生效。如下代码：

```java
public class Test {

    private static Object obj = new Object();

    public static void main(String[] args) throws InterruptedException {
        Runnable run = () -> {
            synchronized (obj) {
                String name = Thread.currentThread().getName();
                System.out.println(name + "进入同步代码块");
                // 保证不退出同步代码块
                try {
                    Thread.sleep(100000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        // 开启两个线程执行同步代码块
        Thread t0 = new Thread(run);
        Thread t1 = new Thread(run);
        t0.start();
        t1.start();
        Thread.sleep(1000);

        // t0线程获得锁，t1处于阻塞状态，向t1发送中断信号
        System.out.println("直接向t1发送中断信号...");
        t1.interrupt();
        Thread.sleep(1000);
        System.out.println("t0线程状态：" + t0.getState());
        System.out.println("t1线程状态：" + t1.getState());

        // 先向t0发送中断信号，再向t1发送中断信号
        System.out.println("\n先向t0发送中断信号，再向t1发送中断信号");
        t0.interrupt();
        Thread.sleep(1000);
        t1.interrupt();
        Thread.sleep(1000);
        System.out.println("t0线程状态：" + t0.getState());
        System.out.println("t1线程状态：" + t1.getState());
    }
}
```

运行结果：

```ASN.1
Thread-0进入同步代码块
直接向t1发送中断信号...
t0线程状态：TIMED_WAITING
t1线程状态：BLOCKED

先向t0发送中断信号，再向t1发送中断信号
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.alanotes.demo.Test.lambda$main$0(Test.java:21)
	at com.alanotes.demo.Test$$Lambda$1/2129789493.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:745)
Thread-1进入同步代码块
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.alanotes.demo.Test.lambda$main$0(Test.java:21)
	at com.alanotes.demo.Test$$Lambda$1/2129789493.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:745)
t0线程状态：TERMINATED
t1线程状态：TERMINATED

Process finished with exit code 0
```

从上面的运行结果可以看出：

当线程 t0 拥有锁，而线程 t1 没有获得锁时，直接向线程 t1 发送中断信号，线程 t1 的状态没有改变，仍然是阻塞状态，说明线程在没有获得锁的阻塞状态下不可被中断；

当先向线程 t0 发送中断信号，再向线程 t1 发送中断信号时，线程 t0 被中断了，后面 t1 也被中断了。这是因为先获得锁的 t0 被中断了，释放了锁，t1 获得了锁，后面又接收到中断信号，也被中断了。

以上结论说明，线程等待获取锁的时候不可被中断，但是获取到锁之后可被中断。

2）我们常说`synchronized`不可以被中断，并不指`synchronized`方法不可中断，比如执行如下代码：

```java
public class Test {
    public synchronized void foo() throws InterruptedException {
        System.out.println("foo begin");
        for (int i =0; i < 100; i++) {
            System.out.println("foo ...");
            Thread.sleep(1000);
        }
    }

    public static void main(String[] args) {
        Test it = new Test();
        ExecutorService es = Executors.newCachedThreadPool();
        es.execute(() -> {
            try {
                it.foo();
            } catch (InterruptedException e) {
                System.out.println("foo is interrupted, msg=" + e.getMessage());
            }
        });
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        es.shutdownNow();  // 关闭线程池，会给线程池中的线程发送中断信号
        System.out.println("Main end");
    }
}
```

运行结果如下：

```ASN.1
foo begin
foo ...
foo ...
foo ...
foo is interrupted, msg=sleep interrupted
Main end

Process finished with exit code 0
```

从上面的运行结果可以看出，当主线程执行`es.shutdownNow()`代码关闭线程池时，子线程被中断了，说明`synchronized`方法可以被中断。但是，当修改 synchronized 方法的代码如下，去掉 throws InterruptedException 声明。

```java
public class Test {
    public synchronized void foo() {
        System.out.println("foo begin");
        for (int i =0; i < 100; i++) {
            System.out.println("foo ...");
            try {
                Thread.sleep(1000);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        Test it = new Test();
        ExecutorService es = Executors.newCachedThreadPool();
        es.execute(() -> {
            try {
                it.foo();
            } catch (Exception e) {
                System.out.println("foo is interrupted, msg=" + e.getMessage());
            }
        });
        
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        es.shutdownNow();
        System.out.println("Main end");
    }
}
```

运行结果如下：

```ASN.1
foo begin
foo ...
foo ...
foo ...
Main end
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.alanotes.demo.Test.foo(Test.java:16)
	at com.alanotes.demo.Test.lambda$main$0(Test.java:27)
	at com.alanotes.demo.Test$$Lambda$1/363771819.run(Unknown Source)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
foo ...
foo ...
foo ...
foo ...
foo ...
foo ...
```

当子线程接收到中断信号后，仍在继续执行。说明`synchronized`方法没有被中断。

根据 Java 文档，判断方法很简单：只要调用的方法抛出 InterruptedException 异常，那么它就可以被中断。不抛出 InterruptedException 的方法是不可中断的。

## 同步原理

synchronized 锁的实现是依赖底层 JVM 的。

- #### synchronized 同步语句块原理


```java
public class SynchronizedDemo {
    public void sync() {
        synchronized (this) {
            // 业务代码
        }
    }
}
```

编写如上测试代码，通过 JDK 自带的`javap`命令查看`SynchronizedDemo`类的相关字节码信息。首先切换到类的对应目录执行`javac SynchronizedDemo.java`命令生成编译后的 .class 类文件，然后执行`javap -c -s -v -l SynchronizedDemo.class`命令。得到如下指令：

![1.png](https://i.loli.net/2021/03/02/3hBe8u5GtRMgS2N.png)

------

从上图可以看出：

`synchronized`同步语句块的实现是使用了`monitorenter`和`monitorexit`指令。其中`monitorenter`指令指向同步代码块的开始位置，`monitorexit`指令指向同步代码块的结束位置。当执行`monitorenter`指令时，线程试图获取锁，也就是获取对象监视器`monitor`的所有权。

在 Java 虚拟机 (HotSpot) 中，每个对象中都内置了一个`monitor`监视器锁。另外，`wait/notify`等方法也依赖于`monitor`对象，这就是为什么只有在同步代码块或者同步方法中才能调用`wait/notify`等方法，否则会抛出`java.lang.IllegalMonitorStateException`异常的原因。

在执行`monitorenter`时，会尝试获取对象的锁，如果锁的计数器为0，则表示锁可以被获取，获取后将锁计数器加1。在执行`monitorexit`指令后，将锁计数器减1，如果计数器的值变为0，表明锁被释放。

如果获取对象锁失败，那当前线程就要阻塞等待，直到锁被另外一个线程释放为止。

另外，从上图可以看出，`monitorexit`指令出现了两次，第1次为同步正常退出释放锁，第2次为发生异常退出释放锁。这其实也是`synchronized`的优点：无论代码执行情况如何，都不会忘记主动释放锁。

- #### synchronized 修饰方法原理


```java
public class SynchronizedDemo {
    public synchronized void sync() {
        // 业务代码
    }
}
```

根据上面同样的步骤，反编译一下：

![2.png](https://i.loli.net/2021/03/02/xCMtDh59rmgjpLl.png)

`synchronized`修饰的方法并没有`monitorenter`指令和`monitorexit`指令，取得代之的确实是`ACC_SYNCHRONIZED`标识，该标识指明了该方法是一个同步方法。JVM 通过该`ACC_SYNCHRONIZED`访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

- #### 总结


`synchronized`同步语句块的实现使用的是`monitorenter`和`monitorexit`指令，其中`monitorenter`指令指向同步代码块的开始位置，`monitorexit`指令则指明同步代码块的结束位置。

`synchronized`修饰方法使用的是`ACC_SYNCHRONIZED`标识，该标识指明了该方法是一个同步方法。

两者的本质都是获取对象监视器`monitor`的所有权。

## 同步概念

- #### Java 对象头


在 JVM 中，对象在内存中的划分为三块区域：对象头、实例数据和对齐填充。`synchronized`用的锁就存在 Java 对象头里。

对象头由两部分组成：

- Mark Word：存储自身的运行时数据，例如 HashCode、GC 年龄、锁相关信息等内容。
- Klass Pointer：类型指针指向它的类元数据的指针。

64 位虚拟机 Mark Word 是 64bit，在运行期间，Mark Word里存储的数据会随着锁标志位的变化而变化。

![1692410-20200429084041363-1585548718.png](https://i.loli.net/2021/03/02/uTwKbdhQt7vp6ex.png)

- #### 监视器（monitor）


任何一个对象都有一个 monitor 与之关联，当 monitor 被获取后，它将处于锁定状态。synchronized 在 JVM 里的实现都是基于进入和退出 monitor 对象来实现方法和代码块同步的，虽然具体实现细节不一样，但是都可以通过成对的 monitorenter 和 monitorexit 指令来实现。

当使用 synchronized 获取对象锁，MarkWord 锁标识位置为10，其中指针指向的是 monitor 对象的起始地址。在 Java 虚拟机（HotSpot）中，monitor 是由ObjectMonitor 实现的。

## 锁优化（锁升级）

因为`monitor`依赖操作系统的 Mutex lock 实现，是一个比较重的操作，需要切换系统至内核态，开销非常大。从 JDK6 开始，对`synchronized`的实现机制进行了较大调整，包括使用 JDK5 引进的 CAS 自旋之外，还增加了自适应的 CAS 自旋、锁消除、锁粗化、偏向锁、轻量级锁这些优化策略。由于`synchronized`关键字的优化使得性能极大提高，同时语义清晰、操作简单、无需手动释放锁，所以推荐尽量使用`synchronized`关键字加锁。

synchronized 有四种状态：无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁。锁可以从偏向锁升级到轻量级锁，再升级到重量级锁。但是锁只能升级，不能降级。

![img](https://gitee.com/sanfene/picgo/raw/master/lock.png)

- #### **无锁**


没有对资源进行锁定，所有线程都能访问和修改，但同时只有一个线程能修改成功。

- #### 偏向锁


偏向锁是 JDK6 中的重要概念，因为经过实践发现，大多数情况下锁被多个线程竞争的情况不多，相反总是由同一线程多次获得，为了让线程获得锁的代价更低，引进了偏向锁。因为轻量级锁的加锁解锁操作是需要依赖多次CAS 原子指令的，而偏向锁只需要在置换 ThreadID 的时候依赖一次CAS原子指令。

当一个线程获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要进行 CAS 操作来加锁和解锁，只需简单地测试一下对象头的里是否存储着指向当前线程的偏向锁。如果存在，表示线程已经获得了锁。

如果不存在，则需要再测试一下 Mark Word 中偏向锁的标识是否设置成1（表示当前是偏向锁）：如果没有设置，则使用 CAS 竞争锁；如果设置了，则尝试使用CAS 将对象头的偏向锁指向当前线程。

偏向锁的撤销，需要等待全局安全点（在这个时间点上没有正在执行的字节码）。它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着， 如果线程不处于活动状态，则将对象头设置成无锁状态；如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的 Mark Word 要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。流程如下图所示：

![3.png](https://i.loli.net/2021/03/02/O9pRe5AIdEWsVmv.png)

偏向锁是在单线程执行代码块时使用的机制，如果在多线程并发的环境下（即线程A尚未执行完同步代码块，线程B发起了申请锁的申请），则一定会转化为轻量级锁或者重量级锁。偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时， 持有偏向锁的线程才会释放锁。

因为偏向锁的撤销操作还是比较重的，在线程竞争资源比较激烈的情况下会影响性能，可以使用`-XX:-UseBiasedLocking=false`禁用偏向锁。

- #### 轻量级锁


引入轻量级锁的主要目的是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。当关闭偏向锁功能或者多个线程竞争偏向锁导致偏向锁升级为轻量级锁，则会尝试获取轻量级锁。

**1）轻量级锁加锁**

线程在执行同步块之前，JVM 会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的 Mark Word 复制到锁记录中，官方称为 Displaced Mark Word。然后线程尝试使用 CAS 将对象头中的 Mark Word 替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。

**2）轻量级锁解锁**

轻量级解锁时，会使用原子的 CAS 操作将 Displaced Mark Word 替换回到对象头。如果成功，则表示没有竞争发生；如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。

下图是两个线程同时争夺锁，导致锁膨胀的流程图：

![4.png](https://i.loli.net/2021/03/02/xaBbmrCEyc9sD3Q.png)

因为自旋会消耗CPU，为了避免无用的自旋（比如获得锁的线程被阻塞住了），一旦锁升级成重量级锁，就不会再恢复到轻量级锁状态。当锁处于这个状态下，其他线程试图获取锁时， 都会被阻塞住，当持有锁的线程释放锁之后会唤醒这些线程，被唤醒的线程就会进行新一轮的夺锁之争。

- #### 锁的优缺点比较


各种锁在不同场景下有不同的选择。每种锁是只能升级，不能降级，即由偏向锁 —> 轻量级锁 —> 重量级锁，而这个过程就是开销逐渐加大的过程。

- 如果是单线程使用，那偏向锁毫无疑问代价最小，并且它就能解决问题，连 CAS 都不用做，仅仅在内存中比较下对象头就可以了；

- 如果出现了其他线程竞争，则偏向锁就会升级为轻量级锁；

- 如果其他线程通过一定次数的 CAS 尝试没有成功，则进入重量级锁；

锁的优缺点如下表：

| 锁       | 优点                                                         | 缺点                                             | 适用场景                           |
| -------- | ------------------------------------------------------------ | ------------------------------------------------ | ---------------------------------- |
| 偏向锁   | 加锁和解锁不需要额外的消耗，和执行非同步方法仅有纳米级的差距 | 如果线程间存在锁的竞争，会带来额外的锁撤销的消耗 | 适用于只有一个线程访问的同步块场景 |
| 轻量级锁 | 竞争的线程不会阻塞，提高了程序的相应速度                     | 如果始终得不到锁竞争的线程，使用自旋会消耗 CPU   | 追求响应时间，同步响应非常快       |
| 重量级锁 | 线程竞争不使用自旋，不会消耗 CPU                             | 线程阻塞，响应时间慢                             | 追求吞吐量，同步块执行时间较长     |