# ThreadLocal 本地局部变量

## 作用

在涉及到多线程需要共享某一资源，但又不能相互影响，必须实现互斥访问时，可以有两种解决思路：

1）使用互斥锁，互斥访问资源。比如`synchronized`关键字，那么在每个时刻只能有一个线程使用该资源，用完后让出位置，其他线程继续使用。这种方式的缺点在于增加了线程间的竞争，降低了效率。

2）为每个线程创建一份资源，访问相互不干预。这样的思路就是将资源进行复制，每个线程使用自己的，各个线程相互不影响，比如使用`ThreadLocal`为每个线程保存一份资源。

如果说`synchronized`是以“时间换空间”，那么`ThreadLocal`就是 “以空间换时间” 。因为`ThreadLocal`的原理就是：对于要在线程间共享的资源或变量，为每个线程都提供一份，从而不同线程之间各自使用各自的，互不影响，从而达到并发访问而不出现问题。

当使用`ThreadLocal`维护变量的时候，为每一个使用该变量的线程提供一个独立的变量副本，即每个线程内部都会有一个该变量，这样同时多个线程访问该变量并不会彼此相互影响，因此他们使用的都是自己从内存中拷贝过来的变量的副本，这样就不存在线程安全问题，也不会影响程序的执行性能。

## 应用实例

这里有个问题，直接看局部变量和`ThreadLocal`起到的作用似乎是一样的，都是保证了并发环境下线程数据的安全性。那是不是完全可以用局部变量来代替`ThreadLocal`？其实不是这样的。`ThreadLocal`提供的是一种线程局部变量，这些变量不同于其它变量的点在于每个线程在获取变量的时候，都拥有它自己相对独立的变量初始化拷贝。

`ThreadLocal`不是为了满足多线程安全而开发出来的，因为局部变量已经足够安全。`ThreadLocal`是为了方便线程处理自己的某种状态。可以看到`ThreadLocal`实例化所处的位置，是一个线程共有区域。好比一个银行和个人，我们可以把钱存在银行，也可以把钱存在家。存在家里的钱是局部变量，仅供个人使用；存在银行里的钱也不是说任何人都可以随便取，也只有我们以个人身份才能取。

- 关于数据库连接的例子

如下是一个数据库链接管理类代码：

```java
class ConnectionManager {
     
    private static Connection connect = null;
     
    public static Connection openConnection() {
        if(connect == null){
            connect = DriverManager.getConnection();
        }
        return connect;
    }
     
    public static void closeConnection() {
        if(connect != null)
            connect.close();
    }
}
```

以上这段代码在单线程中运行使用是没有任何问题的，但是如果在多线程中使用会存在线程安全问题：

1）这里面的2个方法都没有进行同步，很可能在`openConnection`方法中会多次创建`connection`对象；

2）`connection`对象是共享变量，那么在调用`connection`对象的地方需要使用互斥锁来保障线程安全，因为很可能一个线程在使用`connect`进行数据库操作，而另外一个线程调用`closeConnection`关闭了连接。

所以出于线程安全的考虑，可以将这段代码的两个方法进行同步处理。但这样将会大大影响程序执行效率，因为一个线程在使用`connect`进行数据库操作的时候，其他线程只有等待。当然解决这个问题可以有好几种方式：

1）每次调用`openConnection`都创建新的`connection`对象，比如在`ConnectionManager`类中去掉`if(connect == null)`的判断，这样每个线程每次操作都会创建新的`connection`对象，也就不存在线程间共享使用`connection`的问题了。

2）在每个需要使用数据库连接的方法中具体使用时才创建数据库链接，然后在方法调用完毕再释放这个连接。比如下面这样：

```java
public void insert() {
    ConnectionManager connectionManager = new ConnectionManager();
    Connection connection = connectionManager.openConnection();
         
    //使用connection进行操作
        
    connectionManager.closeConnection();
}
```

这样每次都是在方法内部创建连接，方法运行完就关闭，不同`connection`对象使用的周期仅限于方法内，那么线程之间自然也不存在线程安全问题了。但这种方式需要在方法中频繁地开启和关闭数据库连接，这不仅严重影响程序执行效率，还可能导致数据库服务器压力过大。

3）使用`ThreadLocal`为每个线程保存一个`connection`对象的副本，线程之间使用互不影响，这样就不存在线程安全问题，也不会严重影响程序执行性能。

```java
class ConnectionManager {
    // 定义一个用于放置数据库连接的局部线程变量（使每个线程都拥有自己的连接）
    private static ThreadLocal<Connection> connContainer = new ThreadLocal<Connection>();
     
    public static Connection openConnection() {
        Connection conn = connContainer.get();
        try {
            if (conn == null) {
                conn = DriverManager.getConnection();
            }
        } finally {
            connContainer.set(conn);
        }
        return conn;
    }
     
    public static void closeConnection() {
        Connection conn = connContainer.get();
        try {
            if (conn != null) {
                conn.close();
            }
        } finally {
            connContainer.remove();
        }
    }
}
```

## 实现原理

`ThreadLocal`的主要作用是做数据隔离，填充的数据只属于当前线程，变量的数据对别的线程而言是相对隔离的，在多线程环境下，可以防止自己的变量被其它线程篡改。`ThreadLocal`又被叫做线程局部变量，在每个线程中都创建了一个`ThreadLocalMap`对象，每个线程可以访问自己内部`ThreadLocalMap`对象内的`value`。这里需要注意`ThreadLocalMap`，虽然带有`Map`这个词，但是它并没有实现`Map`接口，仅仅是个类名而已，可以说跟`Map`没有半毛钱关系。

那么ThreadLocal是如何实现为每个线程创建变量副本的呢？首先我们来看下`ThreadLocal`的`set`方法。`set`方法在初始化变量值的时候会被调用。

```java
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        }else {
        	//创建ThreadLocalMap
            createMap(t, value);
        }
}

void createMap(Thread t, T firstValue) {
   //当前线程的成员变量threadLocals持有ThreadLocalMap的引用，而ThreadLocal对象（this）仅用来作为key保存和获取值
   t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

当前线程的`ThreadLocal`在第一次调用`set`方法的时候会创建`ThreadLocalMap`并设置上相应的值。每个线程都有一个`threadLocals`成员，引用类型是`ThreadLocalMap`，以`ThreadLocal`和变量值作为参数。这样，我们所使用的`ThreadLocal`变量的实际数据，通过`get`方法取值的时候，就是通过取出`Thread`中`threadLocals`引用的`ThreadLocalMap`，然后从这个`ThreadLocalMap`中根据当前`ThreadLocal`对象作为参数，取出数据。

![5.png](https://i.loli.net/2021/03/04/smBHuvWh29PinDV.png)



也就是说其实不同线程取到的变量副本都是由线程本身保存了，只是借助`ThreadLocal`去获取，不是存放于`ThreadLocal`中。其实也可以这么理解，每个线程各自维护了一个保存有变量副本的`ThreadLocalMap`，而`ThreadLocal`其实只是个符号意义，本身不存储变量副本，仅仅是用来索引各个线程中保存的变量副本数据而已。

![4.png](https://i.loli.net/2021/03/04/Yp8wkiRAmq3TQxc.png)



## 注意事项

- **内存泄露问题**

```
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
    	super(k);
    	value = v;
    }
}
```

`ThreadLocal`在保存的时候会把自己当做 Key 存在线程的`ThreadLocalMap`中，正常情况应该是`key`和`value`都应该被外界强引用才对，但是`key`被设计成`WeakReference`弱引用了。这就导致了一个问题，`ThreadLocal`在没有外部强引用时，发生`GC`时会被回收，如果创建`ThreadLocal`的线程一直持续运行，那么这个`Entry`对象中的`value`就有可能一直得不到回收，发生内存泄露。

在`ThreadLocal`中内存泄漏是指`ThreadLocalMap`中的`Entry`中 的`key`为`null`，而`value`不为`null`。因为`key`为`null`导致`value`一直访问不到，而根据可达性分析，始终有`threadRef -> currentThread -> threadLocalMap -> entry -> valueRef -> valueMemory`，导致在垃圾回收的时候进行可达性分析的时候，`value`可达从而不会被回收掉，但是该value永远不能被访问到，这样就存在了内存泄漏。

因为`Entry`的`key`是弱引用，所以在`gc`的时候`key`会被回收，而`value`是强引用，导致`value`不会被回收。

如果不使用弱引用也会可能会发生内存泄漏，只要在业务代码里，将`ThreadLocal`的引用置为`null`，也会导致`Entry`中`value`访问不到，但又因为可达，所以`gc`时候不会被回收，相当于这部分内存资源被浪费了。

![1.png](https://i.loli.net/2021/03/03/ErVnLgXbWSqodF9.png)

那为什么`Entry`的`key`使用弱引用呢？

假设`ThreadLocal`使用的是强引用，在业务代码中执行`ThreadLocal Instance = null`操作，以清理掉`ThreadLocal`实例的目的，但是因为`ThreadLocalMap`的`Entry`强引用`ThreadLocal`，因此在`gc`的时候进行可达性分析，`ThreadLocal`依然可达，对`ThreadLocal`并不会进行垃圾回收，这样就无法真正达到业务逻辑的目的，出现逻辑错误。假设`Entry`弱引用`ThreadLocal`，尽管会出现内存泄漏的问题，但是在`ThreadLocal`的生命周期里（`set，getEntry，remove`）里，都会针对`key`为`null`的脏`entry`进行处理。
`ThreadLocal`源码中其实已经对内存泄漏问题做了很多优化，在`set，get，remove`方法中都会对`key`为`null`的但是`value`不为`null`的`Entry`进行`value`置`null`操作，使得`value`的引用为`null`，可达性失败，在`gc`是可以回收`value`的内存。

在日常使用中，最后用完`ThreadLocal`后，记得调用`remove`方法。因为如果不调用`remove`方法，当一次`gc`执行，这个`value`就会造成内存泄漏直到当前线程结束（线程结束，`ThreaLocalMap`会被置为null，而`ThreaLocalMap`中的`Entry`也就不可达会被回收，一切都被回收）

线程结束时会执行`Thread.exit`方法

```java
private void exit() {
    if (group != null) {
        group.threadTerminated(this);
        group = null;
    }
    /* Aggressively null out all reference fields: see bug 4006245 */
    target = null;
    /* Speed the release of some of these resources */
    threadLocals = null;
    inheritableThreadLocals = null;
    inheritedAccessControlContext = null;
    blocker = null;
    uncaughtExceptionHandler = null;
}
```

- **重写`initialValue`方法初始化变量值**

先看一个简单的例子，覆盖`ThreadLocal`的`initialValue`方法初始化变量值，如下代码：

```java
public class Test {
        
    //创建一个 Integer 型的线程本地变量
	public static final ThreadLocal<Integer> local = new ThreadLocal<Integer>() {
		@Override
		protected Integer initialValue() {
			return 0;
		}
	};
	public static void main(String[] args) throws InterruptedException {
		Thread[] threads = new Thread[5];
		for (int j = 0; j < 5; j++) {		
               threads[j] = new Thread(new Runnable() {
				@Override
				public void run() {
                    //获取当前线程的本地变量，然后累加5次
					int num = local.get();
					for (int i = 0; i < 5; i++) {
						num++;
					}
                    //重新设置累加后的本地变量
					local.set(num);
					System.out.println(Thread.currentThread().getName() + " : "+ local.get());

				}
			}, "Thread-" + j);
		}

		for (Thread thread : threads) {
			thread.start();
		}
	}
}
```

运行后结果：

```ASN.1
Thread-0 : 5
Thread-2 : 5
Thread-3 : 5
Thread-1 : 5
Thread-4 : 5
```

我们看到，每个线程累加后的结果都是5，各个线程处理自己的本地变量值，线程之间互不影响。现在，我们对`initialValue`方法的返回值稍作改动如下：

```java
public class ThreadLocalTest {
	private static Index num = new Index();
        //创建一个Index类型的本地变量 
	private static ThreadLocal<Index> local = new ThreadLocal<Index>() {
		@Override
		protected Index initialValue() {
			return num;
		}
	};

	public static void main(String[] args) throws InterruptedException {
		Thread[] threads = new Thread[5];
		for (int j = 0; j < 5; j++) {
			threads[j] = new Thread(new Runnable() {
				@Override
				public void run() {
                                        //取出当前线程的本地变量，并累加1000次
					Index index = local.get();
					for (int i = 0; i < 1000; i++) {                                          
						index.increase();
					}
					System.out.println(Thread.currentThread().getName() + " : "+ index.num);

				}
			}, "Thread-" + j);
		}
		for (Thread thread : threads) {
			thread.start();
		}
	}

	static class Index {
		int num;

		public void increase() {
			num++;
		}
	}
}
```

执行后我们发现结果如下（每次运行还都不一样）：

```ASN.1
Thread-0 : 1390
Thread-3 : 2390
Thread-4 : 4390
Thread-2 : 3491
Thread-1 : 1390
```

上面代码中，我们通过覆盖`initialValue`方法来给我们的`ThreadLocal`提供初始值，每个线程都会获取这个初始值的一个副本。而现在我们的初始值是`num`，`num`是这个对象的引用。也就是说我们的初始值是一个引用，引用的副本和引用指向的是同一个对象。

![2.png](https://i.loli.net/2021/03/03/dRo4sjA5qgytU1x.png)

如果我们想给每一个线程都保存一个`Index`对象，就应该创建对象的副本而不是对象引用的副本，比如：

```java
private static ThreadLocal<Index> local = new ThreadLocal<Index>() {
	@Override
	protected Index initialValue() {
		return new Index(); //注意这里
	}
};
```

对象的拷贝图示：

 ![3.png](https://i.loli.net/2021/03/03/KJHlWn5RbNPrU7O.png)

如此运行，结果就正确了。





