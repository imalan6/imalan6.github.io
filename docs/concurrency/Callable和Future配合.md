

# Callable 和 Future/FutureTask

## Callable 接口

线程的创建方式中有两种，一种是实现`Runnable`接口，另一种是继承`Thread`类，但是这两种方式都有个缺点，那就是在任务执行完成之后无法获取返回结果，于是就有了`Callable`接口。`Callable`接口类似于`Runnable`接口，但是`Runnable`不会返回结果，并且无法抛出返回结果的异常，而`Callable`功能更强大一些，可以返回值，它可以和`Future`接口与`FutureTask`类的配合使用以取得线程运行返回的结果。

`Runnable`接口的`run()`方法返回值类型为`void`，所以无法获取结果。

```java
public interface Runnable {  
    public abstract void run();  
} 
```

而`Callable`的接口定义如下：

```java
 public interface Callable<V> {   
    V call() throws Exception;   
 }   
```

该接口声明了一个名称为`call()`的方法，同时这个方法可以有返回值V，也可以抛出异常。

无论是`Runnable`接口还是`Callable`接口的实现类，都可以被`ThreadPoolExecutor`或`ScheduledThreadPoolExecutor`执行，`ThreadPoolExecutor`或`ScheduledThreadPoolExecutor`都实现了`ExcutorService`接口，而因此 Callable 需要和`Executor`框架中的`ExcutorService`结合使用。

`ExecutorService`提供的方法：

```java
<T> Future<T> submit(Callable<T> task);  //提交一个实现了Callable接口的任务，并且返回封装了异步计算结果的Future对象。
<T> Future<T> submit(Runnable task, T result);  //提交一个实现了Runnable接口的任务，指定了在调用Future的get方法时返回的result对象。（不常用）
Future<?> submit(Runnable task);  //提交一个实现了Runnable接口的任务，并且返回封装了异步计算结果的Future。
```
只要创建了`Callable`或者`Runnable`对象，然后通过上面 3 个方法提交给线程池执行即可。另外，除了我们自己实现`Callable`对象外，还可以使用工厂类`Executors`把一个`Runnable`对象封装成`Callable`对象。`Executors`工厂类提供的方法如下：

```java
public static Callable<Object> callable(Runnable task)  
public static <T> Callable<T> callable(Runnable task, T result)  
```

## Future<V> 接口

`Future<V>`接口可以用来获取线程异步计算结果，还可以取消，判断线程任务是否完成等操作。`Future`接口代码如下：

```java
public interface Future {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

`get()`：获取异步执行的结果，如果没有结果可用，此方法会阻塞直到异步计算完成。

`get(Long timeout , TimeUnit unit)`：类似`get()`方法，但是会有等待时间限制，如果阻塞时间超过设定的`timeout`值，该方法将抛出异常。

`isDone()`：如果任务执行结束，无论是正常结束或是中途取消还是发生异常，都返回`true`。

`isCancelled()`：如果任务完成前被取消，则返回`true`。

`cancel (boolean mayInterruptRunning)`：如果任务还没开始，执行`cancel()`方法将返回`false`；如果任务已经启动，执行`cancel(true)`方法将以中断执行此任务线程的方式来试图停止任务，如果停止成功，返回`true`；当任务已经启动，执行`cancel(false)`方法将不会对正在执行的任务线程产生影响，此时返回`false`；当任务已经完成，执行`cancel()`方法将返回`false`。`mayInterruptRunning`参数表示是否中断执行中的线程。

## FutureTask 类

`FutureTask`是实现了`Future`和`Runnable`接口。实现了`Runnable`接口，说明可以把`FutureTask`实例传入到 Thread 中，在一个新的线程中执行。实现`Future`接口，说明可以从`FutureTask`中通过`get`取到任务的返回结果，也可以执行取消任务执行等操作。

`FutureTask`可用于异步获取执行结果或取消执行任务的场景。通过传入`Runnable`或者`Callable`的任务给`FutureTask`，直接调用其`run`方法或者放入线程池执行，之后可以在外部通过`FutureTask`的 get 方法异步获取执行结果，因此，`FutureTask`非常适合用于耗时的计算，主线程可以在完成自己的任务后，再去获取结果。另外，`FutureTask`还可以确保即使调用了多次`run`方法，它都只会执行一次`Runnable`或者`Callable`任务，或者通过`cancel`取消`FutureTask`的执行等。

## 使用实例

- **Callable 配合 Future 使用实例**

`Callable`类任务实现代码

```java
public class Task implements Callable<Integer> {  
      
    private int sum;  
    @Override  
    public Integer call() throws Exception {  
        System.out.println("Callable子线程开始计算");  
          
        for(int i=0 ;i<100;i++){  
            sum += i;  
        }  
        System.out.println("Callable子线程计算结束");  
        return sum;  
    }  
}
```

测试代码

```java
public class CallableTest {

    public static void main(String[] args) {
        //创建线程池  
        ExecutorService es = Executors.newSingleThreadExecutor();
        
        //提交任务并获取执行结果  
        Future<Integer> future = es.submit(new Task());
        
        //关闭线程池，线程执行完才关闭  
        es.shutdown();
        
        try {
            Thread.sleep(1000);
            System.out.println("主线程在执行其他任务");

            if (future.get() != null) {
                //输出获取到的结果  
                System.out.println("future.get()-->" + future.get());
            } else {
                //输出获取到的结果  
                System.out.println("future.get()未获取到结果");
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println("主线程在执行完成");
    }
}
```

运行结果：

```ASN.1
Callable子线程开始计算
主线程在执行其他任务
Callable子线程计算结束
future.get()-->4950
主线程在执行完成
```

- **Callable 配合 FutureTask 使用实例**

只需要对上述代码做部分改动即可

```java
//创建FutureTask  
FutureTask<Integer> futureTask = new FutureTask<>(new Task());  
//执行任务  
es.submit(futureTask);  

// 获取返回结果   
if(futureTask.get() != null){  
    //输出获取到的结果  
    System.out.println("futureTask.get()-->" + futureTask.get());  
}
else{  
    //输出获取到的结果  
    System.out.println("futureTask.get()未获取到结果");  
}  
```

