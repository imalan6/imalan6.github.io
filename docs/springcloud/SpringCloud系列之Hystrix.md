# SpringCloud系列之Hystrix

## 服务雪崩

在一个高度服务化的微服务系统中，一个业务逻辑处理通常会依赖多个服务。 如果其中的某一个服务不可用，就会出现线程池里所有线程都因等待响应而被阻塞，从而造成服务雪崩。这种因服务提供者不可用而导致服务调用者不可用，并将不可用逐渐放大传播的过程，就叫服务雪崩效应。比如程序`bug`、大流量请求、硬件故障、缓存雪崩击穿等。如下是一个分布式系统中服务调用的常见模型：

![2.png](https://i.loli.net/2021/03/24/yA2aeO7NMFkKn8j.png)

在服务提供者不可用时，服务调用者可能会出现大量重试情况，比如用户手动重试操作、代码逻辑重试操作等，这些重试操作最终导致请求量进一步加大。当服务调用者使用的同步调用时，大量等待线程占用了大量系统资源。一旦线程资源被耗尽，服务调用者提供的服务也将处于不可用状态，于是服务雪崩效应就产生了。所以，导致雪崩效应的根本原因就是大量请求线程同步等待造成的资源耗尽。

## 产生原因

导致服务不可用的原因有很多，除了服务提供者不可用之外还有其他因素也可能产生雪崩效应：

- 服务调用者请求量激增，导致系统负载升高。比如异常流量、用户重试、代码逻辑重复等。
- 缓存击穿，缓存雪崩等，导致请求都直接转向数据库。
- 重试机制，比如`RPC`框架的`Retry`次数，每次重试都可能会进一步恶化服务提供者。
- 硬件故障，比如设备出问题，机房断电等情况。

## 解决方案

针对服务调用者自身请求量激增问题，可以采取自动扩容方式应对突发流量，或在负载均衡器上实现限流功能。

针对重试问题，可以减少或关闭重试机制。

针对硬件故障，可以采取多机备份，异地多活等方案。

针对服务不可用而导致的系统雪崩问题，主要有以下解决方案：

- 超时机制

在不做任何处理的情况下，服务提供者不可用会导致消费者请求线程强制等待，而造成系统资源耗尽。加入超时机制，一旦超时，就释放资源。由于释放资源速度较快，一定程度上可以抑制资源耗尽的问题。

- 服务隔离


限制请求核心服务提供者的流量，把流量拦截在核心服务之外，这样可以更好地保证核心服务提供者不出问题。对于一些出问题的服务可以限制流量访问，只分配固定线程资源访问，这样能使整体的资源不至于被出问题的服务耗尽。可以通过线程池+队列的方式，通过信号量的方式进行处理。

- 服务熔断


当远程服务不稳定或网络抖动严重时暂时关闭。实时监测应用，一旦发现一定时间内服务调用的失败次数/失败率达到一定阈值时，服务调用就断开(断路器)。此时，服务请求直接返回，不继续执行原本调用逻辑。断开一段时间后（比如10秒），断路器进入半开状态，此时允许调用一次该服务。如果调用成功，则断路器关闭，服务进入正常调用；如果调用仍然失败，断路器继续回到打开状态。经过一段时间（比如10秒）再进入半开状态，尝试调用服务。通过这种断路器打开—半开—关闭的方式， 可以控制服务调用者是否执行服务调用，从而避免频繁调用失败，浪费系统资源。

- 服务降级


所谓服务降级，就是调用者提前实现一个`fallback`熔断回调，当某个服务熔断无法被调用时，服务调用者就会调用`fallback`直接返回一个缺省值。比如，备用接口/缓存/`mock`数据等。这样业务处理方式更友好。

## Hystrix使用

`Hystrix`是由`Netflix`开源的一个延迟和容错库，提供超时机制、限流、熔断、降级全面的功能实现，主要用于隔离访问远程系统、服务或者第三方库，防止级联失败，从而提升系统的可用性与容错性。

#### 服务隔离

`Hystrix`采用了舱壁隔离技术，来将外部依赖进行资源隔离，进而避免任何外部依赖的故障导致本服务崩溃。舱壁隔离，是说将船体内部空间区隔划分成若干个隔舱，一旦某几个隔舱发生破损进水，水流不会在其间相互流动，如此一来船舶在受损时，依然能具有足够的浮力和稳定性，进而减低立即沉船的危险。`Hystrix`的资源隔离策略有两种，分别为：线程池和信号量。

![5.png](https://i.loli.net/2021/03/24/w2JExNzXjsFQy1S.png)

- **线程池**

在`Hystrix`中，如果不使用线程池隔离策略，而是共用一个线程池的话，服务的调用方式如下图：

![3.png](https://i.loli.net/2021/03/24/ps9EIuzlgw1Ay3Y.png)

这种方法共用一个线程池的方式很容易出现问题，比如：如果方法A请求量很大，调用的服务处理又很慢，那方法A很容易把线程池的线程用完，那方法B和方法C就没有线程可以使用了，只能等待，结果就是超时被熔断。导致这种情况出现，并不是被调用的服务不可用，而是服务调用者根本没有线程去处理这些请求。

为了避免上面这种情况，如果`Hystrix`对每个外部调用都单独配一个线程池，这样即使某个外部调用延迟很严重处理不过来，也只是耗尽自己的那个线程池而已，不会影响其他方法的线程池。这样就使得方法间的调用相互隔离开了，互不影响。如下图：

![4.png](https://i.loli.net/2021/03/24/mYOxg8DE2o5U1tu.png)

`Hystrix`是通过命令模式，将每个类型的业务请求封装成对应的命令请求，比如A方法->服务1Command，B方法->服务1Command，C方法->服务2Command。每个类型的`Command`对应一个线程池。创建好的线程池放入到`ConcurrentHashMap`中，比如：

```java
final static ConcurrentHashMap<String, HystrixThreadPool> threadPools = new ConcurrentHashMap<String, HystrixThreadPool>();
threadPools.put(“hystrix-order”, new HystrixThreadPoolDefault(threadPoolKey, propertiesBuilder));
```

- **信号量**

用于隔离本地代码，利用信号量限制同时运行的线程数量。使用一个原子计数器（或信号量）来记录当前有多少个线程在运行，当请求进来时先判断计数器的数值，若超过设置的最大线程个数则拒绝该请求，若不超过则通行，这时候计数器+1，请求返回成功后计数器-1。

- **区别**

执行线程区别：线程池隔离方式执行依赖代码的线程不是原来的线程；而信号量方式依然是原来的请求线程。

应用场景区别：线程池隔离适合并发量比较大的第三方应用或者接口；而信号量隔离适合并发量不大的内部应用或中间件。

执行效率区别：线程池增加了`cpu`的开销；而信号量不会创建副线程，开销较小。

- **注意**

使用线程池隔离时，在某些业务场景下通过`ThreadLocal`来传递数据会出现问题。因为线程池隔离方式下，`Hystrix`会将请求放入方法的线程池去执行，这时请求线程就由A线程变成B线程了，`ThreadLocal`也就消失了。用信号量没问题的，因为还是原来那个线程在处理。

#### 使用方法

##### 1）命令方式

`hystrix`采用了命令模式，客户端需要继承抽象类`HystrixCommand`并实现其`run`方法。

```java
// HelloHystrixCommand要使用Hystrix功能 
public class HelloHystrixCommand extends HystrixCommand {  
    private final String name; 
    public HelloHystrixCommand(String name) {   
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));     
        this.name = name; 
    } 
    // 如果继承的是HystrixObservableCommand，要重写Observable construct() 
    @Override 
    protected String run() {     
        return "Hello " + name; 
    } 
} 
```

##### 2）注解方式

```
@HystrixCommand(commandKey = "getCompanyInfoById",
                    groupKey = "company-info",
                    threadPoolKey = "company-info",
                    fallbackMethod = "fallbackMethod",
                    threadPoolProperties = {
                        @HystrixProperty(name = "coreSize", value = "30"),
                        @HystrixProperty(name = "maxQueueSize", value = "101"),
                        @HystrixProperty(name = "keepAliveTimeMinutes", value = "2"),
                        @HystrixProperty(name = "queueSizeRejectionThreshold", value = "15"),
                     })
```

- `commandKey`：代表一个接口, 如果不配置，默认是`@HystrixCommand`注解修饰的函数的函数名。
- `groupKey`：代表一个服务，一个服务可能会暴露多个接口。`Hystrix`命令默认的线程划分也是根据命令组来实现。默认情况下，`Hystrix`会让相同组名的命令使用同一个线程池，所以我们需要在创建`Hystrix`命令时为其指定命令组来实现默认的线程池划分。
- `threadPoolKey`：对线程池进行更细粒度的配置，默认等于`groupKey`的值。如果依赖服务中的某个接口耗时较长，需要单独特殊处理，最好单独用一个线程池，这时候就可以配置threadpool key。
- `fallbackMethod`：`@HystrixCommand`注解修饰的函数的回调函数，`@HystrixCommand`修饰的函数必须和这个回调函数定义在同一个类中，因为定义在了同一个类中，所以`fackback method`可以是`public/private`均可。
- 线程池配置：`coreSize`表示核心线程数，`hystrix`默认是10；`maxQueueSize`表示线程池的最大队列大小；`keepAliveTimeMinutes`表示非核心线程空闲时最大存活时间；`queueSizeRejectionThreshold`：该参数用来为队列设置拒绝阈值。通过该参数，即使队列没有达到最大值也能拒绝请求。

使用实例：

```java
@RestController
@RequestMapping("/user/hello")
public class TestHelloController {
    @Autowired
    private RestTemplate restTemplate;
    @GetMapping("/hello")
    @HystrixCommand(
            fallbackMethod = "HelloFallback",// 服务降级方法
            // 使用commandProperties 可以配置熔断的一些细节信息
            commandProperties = {
                    //熔断超时时间2s，类似kv形式
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "2000")
            }
    )
    public int getHi(){
        String url  ="http://.....";
        return restTemplate.getForObject(url, Integer.class);
    }
    // 服务降级方法，服务降级后调用该方法返回，返回值类型需要和原方法一致
    public int HelloFallback(){
        return -1;
    }
}
```
##### 3）与Feign配合使用

通过配置`@FeignClient`注解的`fallback`属性指定一个自定义的`fallback`处理类。

```java
@FeignClient(value = "service-hi", fallback = HiFallback.class)
public interface ITestHi {

    @RequestMapping("/hi")
    public String testHi();
}
```

`HiFallback`需要实现`ITestHi`接口，并且在`Spring`容器中注册`bean`。可以有两种方式：一种直接实现`ITestHi`接口，如下：

```java
@Component
class  HiFallback implements ITestHi{
    @Override
	public String testHi(){
		return "error";
	}
}
```

另一种实现`FallbackFactory<ITestHi>`接口，如下：

```java
@Component
class  HiFallback implements FallbackFactory<ITestHi>{

    @Override
    public ITestHi create(Throwable cause) {
       
    }
}
```

## Hystrix监控

- **单服务监控**

1）添加依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-netflix-hystrix-dashboard</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

2）启动类添加注解`@EnableHystrixDashboard`

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients(basePackages = {"com.landcode.land.service.consumer.service"})
@RibbonClient(name = "service-hi", configuration = RibbonConfig.class)
@EnableCircuitBreaker
@EnableHystrixDashboard // 打开HystrixDashboard
public class ConsumerApplication {

    public static void main(String[] args) {
        // TODO Auto-generated method stub
        SpringApplication.run(ConsumerApplication.class, args);
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

3）测试

启动`hystrix-dashboard`后，输入`http://localhost:8081/hystrix`，出现如下页面：

![1.png](https://i.loli.net/2021/03/24/kwITdDYfLVynRoa.png)

在第一个文本框输入要监控的服务，采用`ip + 端口 + hystrix.stream`格式，比如`http://localhost:8801/actuator/hystrix.stream`，则会跳转到监控页面。如果点击后出现 “`Unable to connect to Command Metric Stream.`” 错误。则按如下方式解决：

在监控服务添加：

```yaml
hystrix:
  dashboard:
    proxy-stream-allow-list: "*"
```

在被监控服务添加配置类：

```java
@Configuration
public class HystrixDashboardConfig {
    @Bean
    public ServletRegistrationBean getServlet() {
        HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
        ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
        registrationBean.setLoadOnStartup(1);
        registrationBean.addUrlMappings("/actuator/hystrix.stream"); //访问路径
        registrationBean.setName("hystrix.stream");
        return registrationBean;
    }
}
```

------

- **Turbine聚合监控数据**

1）创建一个监控服务项目，引入如下依赖：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-turbine</artifactId>
</dependency>
```

2）启动类上配置`@EnableTurbine`注解：

```
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients(basePackages = {"com.landcode.land.service.consumer.service"})
@RibbonClient(name = "service-hi", configuration = RibbonConfig.class)
@EnableCircuitBreaker
@EnableHystrixDashboard 
@EnableTurbine // 打开Turbine
public class ConsumerApplication {

    public static void main(String[] args) {
        // TODO Auto-generated method stub
        SpringApplication.run(ConsumerApplication.class, args);
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

3）添加如下配置，指明从哪些微服务收集监控数据

```yaml
turbine:
  app-config: hi,test
  cluster-name-expression: "'default'"
```

注意`turbine`的配置，这里收集`hi`和`test`服务的日志

4）启动监控项目后，在`hystrix-dashboard中`输入`http://localhost:8088/turbine.stream`,则展示如下聚合结果：

![0.png](https://i.loli.net/2021/03/24/gyeGPKJ8mq17U2n.png)

以上`Turbine`聚合微服务的监控数据，然后在`hystrix-dashboard`展示多个微服务的实时监控数据。但是`Turbine`有它的局限性，比如如果微服务之间无法通信，或者服务没在`Eureka`上注册，则`Turbine`无法收集到微服务的日志。这种情况下，需要借助消息中间件来解决。

