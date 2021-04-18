# SpringCloud系列之Ribbon

## 简介

`Ribbon`是一个由`Netflix`发布的开源的客户端负载均衡器，它可以控制`HTTP`和`TCP`客户端的行为，是`SpringCloud-Netflix`中重要的一环。通过`Ribbon`可以将`Netflix`的中间层服务连接在一起。使用时只需为`Ribbon`配置服务提供者地址列表，`Ribbon`就可基于负载均衡算法计算出要请求的目标服务地址。

`Ribbon`客户端组件提供了一系列完善的配置项，如连接超时、重试等。在配置文件中列出`Load Balancer`后面所有的服务，`Ribbon`会自动的基于某种规则（如简单轮询，随机连接，响应时间加权等）去连接这些服务，也很容易实现自定义的负载均衡算法，只需实现`IRule`接口即可。

![0.png](https://i.loli.net/2021/03/21/QBy3TILZYF6CDEz.png)

`Ribbon`是在客户端来实现负载均衡的访问服务，主要的功能点：

- 服务发现，发现依赖服务的列表

- 服务选择规则，在多个服务中如何选择一个有效服务

- 服务监听，检测失效的服务，剔除失效服务


## 使用方法

#### **1）配置**

- 添加依赖：

由于`spring-cloud-starter-netflix-eureka-client`已经包含`spring-cloud-starter-netfilx-ribbon`，所以无需额外添加依赖。

- 启动类配置：

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients(basePackages = {"com.landcode.land.service.consumer.service"}) // 使用feign调用微服务，如果不使用feign可以不加
@RibbonClient(name = "service-hi", configuration = RibbonConfig.class) // 使用自定义配置
public class UserApplication {

    public static void main(String[] args) {
        // TODO Auto-generated method stub
        SpringApplication.run(UserApplication.class, args);
    }

    // 开启负载均衡
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

如代码所示，只需在`RestTemplate`上添加`LoadBalanced` 注解，即可让整合`Ribbon`。

#### **2）RestTemplate + Ribbon使用**

`RestTemplate`是`Spring Resources`中一个访问第三方`RESTful API`接口的网络请求框架，用来消费`REST`服务的。所以`RestTemplate`的主要方法都与`REST`的`Http`协议的一些方法紧密相连，例如`HEAD`、`GET`、`POST`、`PUT`、`DELETE`和`OPTIONS`等方法，这些方法在`RestTemplate`类对应的方法为`headForHeaders()`、`getForObject()`、`postForObject()`、`put()`和`delete()`等。

```java
@RestController
@RequestMapping("/user")
public class TestHiController {

@Autowired
private RestTemplate restTemplate;

@RequestMapping("/hi")
public String hi () {
    // 使用LoadBalancerClient通过service id获取指定服务
    ServiceInstance serviceInstance = loadBalancerClient.choose("service-hi");
    return restTemplate.getForObject(serviceInstance.getUri().toString() + "/hi", String.class);
}
```
代码中`LoadBalancerClient`为`Ribbon`默认的负载均衡`API`，使用的轮询策略；`loadBalancerClient.choose("consul-provider")`用于获取注册中心服务名称为`service-hi`的服务，再通过`getUri()`获取服务的地址；最后通过`restTemplate.getForObject`调用服务。

#### **3）Feign + Ribbon使用**

`Feign`是一个声明式的`Web Service`客户端。它的出现使开发`Web Service`客户端变得很简单。在`Spring Cloud`中使用`Feign`，可以做到使用`HTTP`请求访问远程服务，就像调用本地方法一样的。详见 [SpringCloud系列之Feign](springcloud/SpringCloud系列之Feign.md)。 

```java
@FeignClient(value = "service-hi")
public interface ITestHi {

    @RequestMapping("/hi")
    public String testHi();
}
```

`Feign`是整合了`Ribbon`，使用方式一样。在`controller`里调用代码如下：

```java
@RestController
@RequestMapping("/user")
public class TestHiController {

    @Autowired
    private ITestHi testHi;

    @RequestMapping("/hi")
    public String testHi() {
        return testHi.testHi();
    }
}
```

## 负载均衡策略

#### **1）负载均衡**

当集群里的1台或者多台服务器出现故障时，剩余没有出现故障的服务器可以保证服务正常使用。而对于客户端如何选择调用哪个服务，则需要负载均衡技术。负载均衡有好几种实现策略，常见的有：

- 随机 (`Random`)
- 轮询 (`RoundRobin`)
- 一致性哈希 (`ConsistentHash`)
- 哈希 (`Hash`)
- 加权（`Weighted`）

#### **2）Ribbon负载均衡策略**

`Ribbon`自带多种负载均衡策略，默认的是轮询策略。分别如下：

![1.png](https://i.loli.net/2021/03/21/d7Wq8bB16fQMhuE.png)

- **配置文件配置策略**

```yaml
# 通过配置文件指定服务的ribbon负载均衡策略为RandomRule
service-hi:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

- **代码配置策略**

为某个服务配置代码：

```java
@Configuration
@RibbonClient(name="service-hi",configuration = RibbonConfig.class)
public class ServiceHiRibbonConfiguration {
}
```

`RibbonConfig`类代码：

```java
@Configuration
public class RibbonConfig {

    //Ribbon提供的负载均衡策略
    @Bean
    public IRule ribbonRule(){
        return new RandomRule();
    }
}
```

新建`RibbonConfig`类，`@RibbonClient`注解指定某个服务，针对该服务的配置有效。`configuration`参数指定为`Ribbon`的负载均衡策略配置类。如果全局配置，则不指定服务名称，使用`@RibbonClients(defaultConfiguration = RibbonConfig.class)`；

当然，也可以不单独创建配置类，像开头那样直接放入启动类中也是可以的。

- **自定义策略**

如果`Ribbon`提供的负载均衡策略不满足业务需求，想要自己实现的话也是可以的。`Ribbon`支持自定义负载均衡策略，需要继承`AbstractLoadBalancerRule`类，或者实现`IRule`接口，实现`choose`方法即可。

```java
public class TestRule extends AbstractLoadBalancerRule {
    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {
    }

    @Override
    public Server choose(Object o) {
        log.info("key:" + o);
        List<Server> allServers = getLoadBalancer().getAllServers(); // 获取所有服务实例
        return allServers.get(0); //使用列表中第一个服务实例
    }
}
```

如上，实现了一个简单的测试策略，即调用服务列表中的第一个服务实例。在真实应用环境中，替换成自己实现的负载均衡算法即可。使用该策略的话，直接将配置类`return`自定义的策略就可以了。

```java
@Configuration
public class RibbonConfig {

    @Bean
    public IRule ribbonRule(){
        return new TestRule(); // 返回自定义的策略
    }
}
```









































