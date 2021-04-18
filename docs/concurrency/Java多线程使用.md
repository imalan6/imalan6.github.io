# Java多线程编程

![src_http___attach.bbs.miui.com_forum_201205_03_01400598djmyeczcskh2yr.jpg_refer_http___attach.bbs.miui.jpg](https://i.loli.net/2021/03/02/H1ZBXFaqveYuxim.jpg)

## 多线程好处

为了解决负载均衡问题，充分利用CPU资源。为了提高CPU的使用率，采用多线程的方式去同时完成几件事情而不互相干扰。为了处理大量的IO操作时或处理的情况需要花费大量的时间等等，比如：读写文件，视频图像的采集，处理等。

使用多线程的好处：
- 使用线程可以把占据时间长的程序中的任务放到后台去处理。
- 用户界面更加吸引人，这样比如用户点击了一个按钮去触发某件事件的处理，可以弹出一个进度条来显示处理的进度。
- 程序的运行效率可能会提高。
- 在一些等待的任务实现上如用户输入，文件读取和网络收发数据等，多线程就比较有用了。

## 线程的生命周期

线程的生命周期包含5个阶段，包括：新建、就绪、运行、阻塞、销毁。

- 新建：就是刚使用`new`方法，创建出来的线程；
- 就绪：就是调用线程的`start()`方法后，这时候线程处于等待 CPU 分配资源阶段，谁先抢到 CPU 资源，谁开始执行;
- 运行：当就绪的线程被调度并获得 CPU 资源时，便进入运行状态，run 方法定义了线程的操作和功能;
- 阻塞：在运行状态的时候，可能因为某些原因导致运行状态的线程变成了阻塞状态，比如`sleep()`、`wait()`之后线程就处于了阻塞状态，这个时候需要其他机制将处于阻塞状态的线程唤醒，比如调用`notify`或者`notifyAll()`方法。唤醒的线程不会立刻执行`run`方法，它们要再次等待 CPU 分配资源进入运行状态;
- 销毁：如果线程正常执行完毕后或线程被提前强制性的终止或出现异常导致结束，那么线程就要被销毁，释放资源;

完整的线程生命周期图如下：

![1223046-20190722214114154-276488899.png](https://i.loli.net/2021/02/27/pZy3CdqFn5fK8wU.png)

Java划分更细一点的线程的状态在`java.lang.Thread.State`定义中有6种。

- New，线程被创建，未执行和运行的时候。

- Runnable，不代表线程在跑，分为两种：被 cpu 执行的线程，随时可以被 cpu 执行的状态。

- Blocked，线程阻塞，处于`synchronized`同步代码块或方法中被阻塞。

- Waiting，等待的线程状态。线程当前不执行，如果被其他唤醒后会继续执行的状态。依赖另一个线程的通知。这个等待是一直等，没其他线程唤醒起不来。

- Time Waiting，指定等待时间的等待线程的线程状态。带超时的方式：`Thread.sleep`，`Object.wait`，`Thread.join`，`LockSupport.parkNanos`，`LockSupport.parkUntil`。

- Terminated，正常执行完毕或者出现异常终止的线程状态。

详细状态转换图如下：

![u_960461747,3849268958_fm_26_gp_0.jpg](https://i.loli.net/2021/02/27/Sktb8HpuPvI3oG1.jpg)

## 创建线程的方式

- 继承 Thread 类

用户的线程类只须继承`Thread`类并重写其`run()`方法即可，通过调用用户线程类的`start()`方法启动用户线程

```java
class MyThread extends Thread{
    public void run(){

  	}
}

public class TestThread{
  	public static void main(String[] args）{
      	MyThread thread = new MyThread(); //创建用户线程对象
      	thread.start(); //启动用户线程
      	thread.run(); //主线程调用用户线程对象的run()方法
  	}
}
```

- 实现 Runnable 接口

当使用`Thread(Runnable thread)`方式创建线程对象时，须为该方法传递一个实现了`Runnable`接口的对象，这样创建的线程将调用实现`Runnable`接口的对象的`run()`方法

```java
public class TestThread{
  	public static void main(String[] args){
      	Mythread mt = new Mythread();
      	Thread t = new Thread(mt);  //创建用户线程
       	t.start();  //启动用户线程
  	}
}

class Mythread implements Runnable{
    public void run(){
		// 实现线程业务处理逻辑
    }
}
```

至于哪个好，肯定是后者好，因为实现接口的方式比继承类的方式更灵活，也能减少程序之间的耦合度，是面向接口编程。

## 线程安全

指在并发的情况之下，该代码经过多线程使用，线程的调度顺序不影响任何结果。

线程安全也是有几个级别的：

- 不可变

  像`String`、`Integer`、`Long`这些，都是`final`类型的类，任何一个线程都改变不了它们的值，要改变除非新创建一个，因此这些不可变对象不需要任何同步手段就可以直接在多线程环境下使用。

- 绝对线程安全

  不管运行时环境如何，调用者都不需要额外的同步措施。要做到这一点通常需要付出许多额外的代价，Java中标注自己是线程安全的类，实际上绝大多数都不是线程安全的，不过绝对线程安全的类，Java中也有，比方说`CopyOnWriteArrayList`、`CopyOnWriteArraySet`。

- 相对线程安全

  相对线程安全也就是我们通常意义上所说的线程安全，像`Vector`这种，`add`、`remove`方法都是原子操作，不会被打断，但也仅限于此，如果有个线程在遍历某个`Vector`、有个线程同时在`add`这个`Vector`，99%的情况下都会出现`ConcurrentModificationException`，也就是`fail-fast`机制。

- 线程非安全

  这个就没什么好说的了，`ArrayList`、`LinkedList`、`HashMap`等都是线程非安全的类。

## 锁的分类

死锁：死锁是指两个或两个以上的进程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。

锁的分类：

- 乐观锁/悲观锁
- 独享锁/共享锁
- 互斥锁/读写锁
- 可重入锁
- 公平锁/非公平锁
- 分段锁
- 偏向锁/轻量级锁/重量级锁
- 自旋锁

以上都是一些锁的名词，这些分类并不是完全指锁的状态，有的指锁的特性，有的指锁的设计，详见 [Java锁的分类和使用](concurrency/Java锁的分类和使用)。

## 线程同步

- 线程间通信：


多个线程处理同一个资源，需要线程间通信解决线程对资源的占用，避免对同一资源争夺。及引入等待唤醒机制（`wait()`，`notify()`）

**wait 方法**：线程调用`wait()`方法，释放它对锁的拥有权，然后等待另外的线程来通知它（通知的方式是`notify()`或者`notifyAll()`方法），这样它才能重新获得锁的拥有权和恢复执行。要确保调用`wait()`方法的时候拥有锁，即`wait()`方法的调用必须放在`synchronized`方法或`synchronized`代码块中。

**notify 方法**：`notify()`方法会唤醒一个等待当前对象的锁的线程。唤醒在此对象监视器上等待的单个线程。

**notifyAll 方法**：`notifyAll()`方法会唤醒在此对象监视器上等待的所有线程。

- 线程间共享数据：


方式一：当每个线程执行的代码相同时，可以使用同一个`Runnable`对象

```java
public class MultiThreadShareData {
    public static void main(String[] args) {
        ShareData task = new ShareData();  //一个类实现了Runnable接口
        for(int i = 0; i < 4; i ++) {   //四个线程来卖票
            new Thread(task).start();
        }
    }
}
class ShareData implements Runnable {
    private int data = 100;
    @Override
    public void run() {  //卖票，每次一个线程进来，先判断票数是否大于0
         synchronized(this) {
             if(data > 0) {
                 log.info(Thread.currentThread().getName() + ": " + data);
                 data--;
             }
         }
    }
}
```

方式二：若每个线程执行任务不同，可以将两个任务方法放到一个类中，然后将 data 也放在这个类中，然后传到不同的 Runnable 中，即可完成数据的共享。

```java
public class MultiThreadShareData {
    public static void main(String[] args) {
        ShareData task = new ShareData(); //公共数据和任务放在task中
        for(int i = 0; i < 2; i ++) { //开启两个线程增加data
            new Thread(new Runnable() {
                @Override
                public void run() {
                    task.increment();
                }
            }).start();
        }
        for(int i = 0; i < 2; i ++) { //开启两个线程减少data
            new Thread(new Runnable() {
                @Override
                public void run() {
                    task.decrement();
                }
            }).start();
        }
    }
}

class ShareData implements Runnable {
    private int data = 0;
    public synchronized void increment() { //增加data
        log.info(Thread.currentThread().getName() + ": before : " + data);
        data++;
        log.info(Thread.currentThread().getName() + ": after : " + data);
    }
    public synchronized void decrement() { //减少data
        log.info(Thread.currentThread().getName() + ": before : " + data);
        data--;
        log.info(Thread.currentThread().getName() + ": after : " + data);
    }
}
```

## 线程池

避免频繁地创建和销毁线程带来性能开销，从而达到线程对象的重复利用。同时，使用线程池还可以根据项目灵活地控制并发线程的数量。

- **ThreadPoolExecutor 类**

ThreadPoolExecutor 类是线程池中最核心的一个类，它提供了四个构造方法。

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    /**
    *corePoolSize：核心池的大小
    *maximumPoolSize：线程池最大线程数
    *keepAliveTime：表示线程没有任务执行时最多保持多久时间会终止
    *unit：参数keepAliveTime的时间单位，有7种取值，在TimeUnit类中有7种静态属性
    *    TimeUnit.DAYS;               //天
    *    TimeUnit.HOURS;             //小时
    *    TimeUnit.MINUTES;           //分钟
    *    TimeUnit.SECONDS;           //秒
    *    TimeUnit.MILLISECONDS;      //毫秒
    *    TimeUnit.MICROSECONDS;      //微妙
    *    TimeUnit.NANOSECONDS;       //纳秒
    *workQueue：一个阻塞队列，用来存储等待执行的任务
    *    ArrayBlockingQueue;
    *    LinkedBlockingQueue;
    *    SynchronousQueue;
    *threadFactory：线程工厂，主要用来创建线程
    *handler：表示当拒绝处理任务时的策略，有以下四种取值
    *    ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。
    *    ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。
    *    ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
    *    ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务
    */
    .....
    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
            BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);

    public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,
        BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
    ...
}
```

`ThreadPoolExecutor`的其他方法：

1）`execute()`方法实际上是`Executor`中声明的方法，在`ThreadPoolExecutor`进行了具体的实现，这个方法是`ThreadPoolExecutor`的核心方法，通过这个方法可以向线程池提交一个任务，交由线程池去执行。

2）`submit()`方法是在`ExecutorService`中声明的方法，在`AbstractExecutorService`就已经有了具体的实现，在`ThreadPoolExecutor`中并没有对其进行重写，这个方法也是用来向线程池提交任务的，但是它和`execute()`方法不同，它能够返回任务执行的结果，去看 submit() 方法的实现，会发现它实际上还是调用的`execute()`方法，只不过它利用了`Future`来获取任务执行结果。

3）`shutdown()`和`shutdownNow()`是用来关闭线程池的。

4）还有很多其他的方法：比如：`getQueue() 、getPoolSize() 、getActiveCount()、getCompletedTaskCount()`等获取与线程池相关属性的方法。

- **线程池使用实例**

创建线程池时，并不提倡直接使用`ThreadPoolExcutor`类创建，而是使用`Executors`类中的几个静态方法来创建。而通过`Executors`类的源码可以看出，其实Executors 类也是调用了`ThreadPoolExcutor`类来创建线程池的，比如：

```java
1 Executors.newCachedThreadPool(int Integer.MAX_VALUE);    //创建一个缓冲池，缓冲池容量大小为
2 Executors.newSingleThreadExecutor();   //创建容量为1的缓冲池
3 Executors.newFixedThreadPool();    //创建固定容量大小的缓冲池
```

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}

public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, 
    								TimeUnit.SECONDS, new SynchronousQueue<Runnable>());
}
```

使用示例：

```java
public class Test{
    public static void main(String[] args){
        // 创建一个容量为5的线程池
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        // 向线程池提交一个任务（其实就是通过线程池来启动一个线程）
        for( int i = 0; i<15; i++){
            executorService.execute(new Task());
            log.info("================");
        }
        executorService.shutdown();
    }
}

class Task extends Thread{
    @Override
    public void run(){
        try{
            Thread.sleep(1000);
        }catch(InterruptedException e){
            e.printStackTrace();
        }
    }
}
```

## 多线程常见问题

- **上下文切换**

多线程并不一定是要在多核处理器才支持的，就算是单核也是可以支持多线程的。 CPU 通过给每个线程分配一定的时间片，由于时间非常短通常是几十毫秒，所以 CPU 可以不停的切换线程执行任务从而达到了多线程的效果。

但是由于在线程切换的时候需要保存本次执行的信息，在该线程被 CPU 剥夺时间片后又再次运行恢复上次所保存的信息的过程就称为上下文切换。上下文切换是非常耗效率的。

通常有以下解决方案:

采用无锁编程，比如将数据按照`Hash(id)`进行取模分段，每个线程处理各自分段的数据，从而避免使用锁。

采用 CAS (`compare and swap`) 算法，如`Atomic`包就是采用`CAS`算法。

合理的创建线程，避免创建了一些线程但其中大部分都是处于`waiting`状态，因为每当从`waiting`状态切换到`running`状态都是一次上下文切换。

- **死锁**

死锁的场景一般是：线程 A 和线程 B 都在互相等待对方释放锁，或者是其中某个线程在释放锁的时候出现异常如死循环之类的。这时就会导致系统不可用。

常用的解决方案如下：

尽量一个线程只获取一个锁，一个线程只占用一个资源。

尝试使用定时锁，至少能保证锁最终会被释放。

所有线程按指定顺序获取锁。

- **资源限制**

当在带宽有限的情况下一个线程下载某个资源需要 1M/S，当开 10 个线程时速度并不会乘 10 倍，反而还会增加时间，毕竟上下文切换比较耗时。

如果是受限于资源的话可以采用集群来处理任务，不同的机器来处理不同的数据，就类似于开始提到的无锁编程。