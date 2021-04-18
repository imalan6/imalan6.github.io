# Java回调机制

## 调用关系

模块之间的调用的方式分为如下几种：

**1）同步调用**

![0.png](https://i.loli.net/2021/03/17/ChYzEskBGMZbf4i.png)

同步调用是最基本且最简单的一种调用方式，类A的方法`a()`调用类B的方法`b()`，一直等待`b()`方法执行完毕，`a()`方法继续往下执行。这种调用方式适用于方法`b()`执行时间不长的情况，因为`b()`方法执行时间一长或者直接阻塞的话，`a()`方法的剩余代码是无法执行的，这样会导致整个流程的阻塞。

**2）异步调用**

![1.png](https://i.loli.net/2021/03/17/Zgt2Iiz69EWbOMA.png)

异步调用是为了解决同步调用可能出现的阻塞，导致整个流程卡住而产生的一种调用方式。类A的方法`a()`开启一个新线程调用类B的方法`b()`，方法`a()`继续往下执行，这样无论方法`b()`执行多久，都不会阻塞方法`a()`的执行。但这种方式，由于方法`a()`不等待方法`b()`的执行完成，在方法`a()`需要方法`b()`执行结果的情况下，必须通过一定的方式对方法`b()`的执行结果进行监听，可以使用`Future+Callable`的方式实现。

**3）回调机制**

 ![2.png](https://i.loli.net/2021/03/17/NZmBuEHX3serxRz.png)

回调的思想是：

- 类A的`a()`方法调用类B的`b()`方法

- 类B的`b()`方法执行完毕主动调用类A的`callback()`方法

这样形成了一种双向的调用方式。

## 回调机制

接下来看一下回调的代码示例，代码模拟一个主服务开始执行任务，然后调用计算服务进行计算，计算服务完成后将结果回调给主服务。

首先定义一个回调接口`ServiceCallback`，一个`getResult`方法用于获取计算结果：

```java
public interface ServiceCallback {
	// 用于获取计算结果
    public void getResult(int result);
}
```

定义一个主服务，实现`ServiceCallback`回调接口：

```java
public class MainService implements ServiceCallback {

    public void task(){
        System.out.println("主服务调用计算服务");
        CalculateService calculateService = new CalculateService();
        calculateService.add(1, 5, this);
        
        System.out.println("主服务继续执行其他任务");
    }

    @Override
    public void getResult(int result) {
        System.out.println("主服务获取了计算结果:" + result);
    }

    public static void main(String[] args) {
        MainService mainService = new MainService();
        mainService.task();
    }
}
```

定义一个计算服务：

```java
public class CalculateService {

    public void add(int x, int y, ServiceCallback callback){

        int result = x + y;
        try {
            Thread.sleep(3000);  // 模拟耗时计算
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        System.out.println("计算任务完成了计算");

        // 调用主服务的回调接口，回送计算结果
        callback.getResult(result);
    }
}

```

定义一个测试类：

```java
public class CalculateTest{
	public static void main(String[] args) {
        MainService mainService = new MainService();
        // 执行任务
        mainService.task();
    }
}
```

输出结果：

```ASN.1
主服务调用计算服务
计算任务完成了计算
主服务获取到了计算结果:6
主服务继续执行其他任务
```

回调机制实现步骤：

1）定义一个回调接口，里面方法用于传输执行结果；

2）调用方实现回调接口，将自己(`this`)传给被调用方；

3）被调用方执行完任务，通过调用回调接口`callback`将结果返回给调用方。

总之，**回调的核心就是调用方将本身即this传递给被调用方**，这样被调用方就可以在调用完毕之后返回调用方结果。回调是一种思想、是一种机制，至于具体如何实现，可以根据不同业务做不同地处理。

## 异步回调

从上面回调实例的运行结果可以看出，主服务调用了计算服务后，并没有继续执行后面的代码，而是等待计算服务执行完毕返回结果后，才开始继续向下执行的，这是同步回调。而要实现异步回调的话，需要开启一个新线程来执行计算任务，其他并无差异。

首先，定义一个Task类实现Runnable接口，代码如下：

```java
class Task implements Runnable{

    private ServiceCallback callback;

    public Task(ServiceCallback callback){
        this.callback = callback;
    }

    @Override
    public void run() {
        CalculateService calculateService = new CalculateService();
        calculateService.add(1, 5, callback);
    }
}
```

然后，主服务调用计算服务的task方法改为开启新线程调用，代码如下：

```java
public void task(){
	System.out.println("主服务调用计算服务");
    new Thread(new Task(this)).start();
    System.out.println("主服务继续执行其他任务");
}
```

再次执行后，输出结果：

```ASN.1
主服务调用计算服务
主服务继续执行其他任务
计算任务完成了计算
主服务获取到了计算结果:6
```

由结果可以看出，主服务调用计算服务后，并不是等待执行结果，而是继续向下执行。

