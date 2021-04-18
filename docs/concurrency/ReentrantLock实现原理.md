# ReentrantLock 实现原理

使用`synchronized`来做同步处理时，锁的获取和释放都是隐式的，实现的原理是通过编译后加上不同的机器指令来实现。

而`ReentrantLock`就是一个普通的类，它是基于`AQS(AbstractQueuedSynchronizer)`来实现的。

是一个重入锁：一个线程获得了锁之后仍然可以反复的加锁，不会出现自己阻塞自己的情况。

> `AQS`是`Java`并发包里实现锁同步的一个重要的基础框架。

## 锁类型

`ReentrantLock`分为公平锁和非公平锁，可以通过构造方法来指定具体类型：

```java
    //非公平锁，默认
    public ReentrantLock() {
        sync = new NonfairSync();
    }
    
    //公平锁
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

默认使用非公平锁，它的效率和吞吐量都比公平锁高很多。

## 锁使用

通常的使用方式如下:

```java
    private ReentrantLock lock = new ReentrantLock();
    public void run() {
        lock.lock();
        try {
            // 业务处理
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
```

特别注意，`lock`不同于`synchronized`关键字，需要手动释放。为了防止发生异常导致锁无法释放，通常在`finally`代码块中释放锁。如果要指定公平锁，只需要在构造方法参数中传入`true`即可。

- #### 公平锁

**1、获取锁**

```java
static final class FairSync extends Sync {
    final void lock() {
        acquire(1);
    }
    // AbstractQueuedSynchronizer.acquire(int arg)
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 和非公平锁相比，这里多了一个判断：是否有线程在等待
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

实现步骤：

1）调用 `tryAcquire(arg)`尝试获取锁，判断`AQS`中的锁状态`state`是否等于 0。当`state > 0`时，表示锁已经被获取；当`state == 0`时，表示锁已经释放。

2）如果`state == 0`，表示目前没有线程获得锁，那当前线程就可以尝试获取锁；

3）尝试之前会利用`hasQueuedPredecessors()`方法判断`AQS`的队列中是否有其他线程，如果有则不会尝试获取锁，这是公平锁特有的情况。

4）如果队列中没有线程就利用`CAS`（`compareAndSetState(0, acquires)`方法）来将`AQS`中的`state`置为1，也就是获取锁，获取成功则调用`setExclusiveOwnerThread(current)`将当前线程置为获得锁的独占线程。

如果第 2）步中的`state > 0`时，说明锁已经被获取了，则判断获取锁的线程是否为当前线程(`ReentrantLock`支持重入)，如果是则将`state + 1`。

6）如果获取锁失败，则将当前线程写入队列。

**2、写入队列**

如果`tryAcquire(arg)`获取锁失败，则调用`addWaiter(Node.EXCLUSIVE)`将当前线程封装一个`Node`对象(`addWaiter(Node.EXCLUSIVE)`) 并写入队列中。

> AQS 中的队列是由 Node 节点组成的双向链表实现的。

node 节点封装代码如下：

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

首先判断队列是否为空，不为空时则将封装好的`Node`利用`CAS`写入队尾，如果出现并发写入失败就需要调用`enq(node)`来写入。

```java
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

这个处理方式就相当于`CAS`加上自旋操作（循环等待）保证一定能写入队列。

**3、挂起等待线程**

写入队列之后，调用`acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`将当前线程挂起。

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

首先会根据`node.predecessor()`获取到上一个节点是否为头节点，如果是则再尝试获取一次锁。如果不是头节点，则会根据上一个节点的`waitStatus`状态来处理`shouldParkAfterFailedAcquire(p, node)`。

- #### 非公平锁


公平锁与非公平锁的差异主要在获取锁上。公平锁会让线程依次排队获取锁，而非公平锁则没有这些规则，属于抢占模式，每来一个线程，都会直接先尝试获取锁。

```java
static final class NonfairSync extends Sync {
    final void lock() {
        // 和公平锁相比，这里会直接先进行一次CAS，成功就返回了
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
    // AbstractQueuedSynchronizer.acquire(int arg)
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
/**
 * Performs non-fair tryLock.  tryAcquire is implemented in
 * subclasses, but both need nonfair try for trylock method.
 */
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 这里也是直接CAS，没有判断前面是否还有节点。
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

非公平锁的实现在刚进入`lock`方法时会直接使用一次`CAS`去尝试获取锁，不成功才会到`acquire`方法中，当进入到非公平锁的`nonfairTryAcquire`方法中也并没有判断是否有前驱节点在等待锁，直接`CAS`尝试获取锁。由此实现了非公平锁。

- #### **小结**

非公平锁和公平锁的不同处：

1、非公平锁在调用`lock`后，首先就会调用`CAS`进行一次抢锁，如果这个时候恰巧锁没有被占用，那么直接就获取到锁返回了。

2、非公平锁在`CAS`失败后，和公平锁一样都会进入到`tryAcquire`方法，在`tryAcquire`方法中，如果发现锁这个时候被释放了（`state == 0`），非公平锁会直接`CAS`抢锁，但是公平锁会判断等待队列是否有线程处于等待状态，如果有则不去抢锁，排到队列后面。

公平锁和非公平锁就这两点区别，如果这两次`CAS`都不成功，那么后面非公平锁和公平锁是一样的，都要进入到阻塞队列等待唤醒。

相对来说，非公平锁会有更好的性能，但可能导致阻塞队列中的等待线程长期处于饥饿状态。

## 锁释放

公平锁和非公平锁的释放流程都是一样的：

```java
    public void unlock() {
        sync.release(1);
    }
    
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                   // 唤醒被挂起的线程
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    
    // 尝试释放锁
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }        
```

首先会判断当前线程是否为获得锁的线程，由于是重入锁所以需要将`state`减到 0 才认为完全释放锁。释放之后需要调用`unparkSuccessor(h)`来唤醒被挂起的线程。

## 线程同步（与Condition）

在使用`synchronized`锁时，我们可以使用`notify()`和`wait()`方法实现线程同步。但是当使用`ReentrantLock`锁时，线程同步操作需要借助`Condition`来实现。`Condition`必须和`Lock`配合使用，其提供的`await()`方法和`signal()`方法，作用类似于`Object`对象的`wait()`和`notify()`方法，用来实现线程间的通信。

`Condition`的作用是对锁进行更加精确的控制，`Condition`中的`await() / signal() / signalAll()`和 Object 中的`wait() / notify() / notifyAll()`方法相似，不同的是`Object`中的三个方法是基于`synchronized`关键字，而`Condition`中的三个方法则需要和`Lock`配合使用。针对一个`Lock`对象可以有多个`Condition`对象，这使得`Condition`对象更加灵活。

一个经典的生产者 - 消费者模型例子，代码如下：

```java
public class Repository {
    /*
     * 仓库的容量
     */
    private int capacity;
    /*
     * 仓库的实际量
     */
    private int size;
    /*
     * 独占锁
     */
    private Lock lock;
    /*
     * 仓库满时的条件
     */
    private Condition fullCondition;
    /*
     * 仓库空时的条件
     */
    private Condition emptyCondition;

    public Repository(int capacity) {
        this.capacity = capacity;
        this.size = 0;
        lock = new ReentrantLock();
        fullCondition = lock.newCondition();
        emptyCondition = lock.newCondition();
    }

    class Producer implements Runnable {

        @SneakyThrows
        @Override
        public void run() {

            while (true) {

                //上锁
                lock.lock();
                try {

                    // 随机生成1~10个
                    int num = new Random().nextInt(10) + 1;

                    if (size >= capacity) {
                        System.out.println("repository fulled!");

                        // 仓库满了，相当于wait()操作，只有当fullCondition.signalAll()才能继续执行
                        fullCondition.await();
                    }

                    /*
                     * 获取实际的生产量
                     */
                    int actNum = (size + num) > capacity ? capacity - size : num;
                    size += actNum;

                    System.out.println("produced: +" + actNum + ", the size is:" + size);

                    // 通知消费者来消费，相当于notifyAll()操作
                    emptyCondition.signalAll();

                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    //释放锁一般放在finally中
                    lock.unlock();
                }

//                Thread.sleep(1000);
            }
        }
    }


    class Consumer implements Runnable {

        @SneakyThrows
        @Override
        public void run() {

            while (true) {

                //上锁
                lock.lock();
                try {

                    // 随机生成1~10个
                    int num = new Random().nextInt(10) + 1;

                    if (size < 1) {
                        System.out.println("repository emptied!");

                        // 仓库空了，相当于wait()操作，emptyCondition.signalAll()才能继续执行
                        emptyCondition.await();
                    }

                    /*
                     * 获取实际的消费量
                     */
                    int actNum = (size < num) ? size : num;
                    size -= actNum;

                    System.out.println("consumed: -" + actNum + ", the size is:" + size);

                    // 通知生产者生产，相当于notifyAll()操作
                    fullCondition.signalAll();

                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    //释放锁一般放在finally中
                    lock.unlock();
                }

//                Thread.sleep(1000);
            }
        }
    }

    public static void main(String[] args){
        Repository repository = new Repository(10);

        new Thread(repository.new Producer()).start();

        new Thread(repository.new Consumer()).start();
    }
```

以上代码，首先定义了一个仓库类`repository`，类中定义了一个锁对象`lock`，为生产和消费操作上锁，相当于`synchronized`的作用。然后，由一个锁对象创建出两个条件对象：`fullCondition`（仓库满）和`emptyCondition`（仓库空）。

当生产者生产数据时，调用`lock.lock()`为生产操作上锁，当仓库满了时，执行`fullCondition.await()`，相当于`wait()`作用，只有执行`fullCondition.signalAll()`才能让当前生产线程继续执行。当生产数量没有达到仓库最大容量时，会调用`emptyCondition.signalAll()`通知消费者消费数据。

当消费者消费数据时，同样调用`lock.lock()`为消费操作上锁，当仓库为空时，会执行`emptyCondition.await()`，让消费者线程阻塞，只有生产线程调用`emptyCondition.signalAll()`才能让当前线程继续执行。当消费，会调用`fullCondition.signalAll()`通知生产者产生数据；

最后，当线程的任务执行完毕后，需要调用`lock.unlock()`释放对象锁。

代码运行结果如下：

```ASN.1
produced: +6, the size is:6
produced: +4, the size is:10
repository fulled!
consumed: -6, the size is:4
consumed: -4, the size is:0
repository emptied!
produced: +7, the size is:7
produced: +3, the size is:10
repository fulled!
consumed: -3, the size is:7
consumed: -6, the size is:1
consumed: -1, the size is:0
repository emptied!
produced: +9, the size is:9
produced: +1, the size is:10
repository fulled!
consumed: -6, the size is:4
consumed: -2, the size is:2
consumed: -2, the size is:0
repository emptied!
produced: +3, the size is:3
produced: +6, the size is:9
produced: +1, the size is:10
repository fulled!
consumed: -6, the size is:4
consumed: -4, the size is:0
repository emptied!
produced: +7, the size is:7
produced: +3, the size is:10
repository fulled!
consumed: -7, the size is:3
consumed: -3, the size is:0
repository emptied!
produced: +1, the size is:1
produced: +8, the size is:9
produced: +1, the size is:10
```


