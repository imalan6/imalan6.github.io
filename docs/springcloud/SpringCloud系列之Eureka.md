# SpringCloud系列之Eureka

## 简介

- **服务治理**

服务治理是微服务架构中最为核心和基础的模块，它主要用来实现各个微服务实例的自动化注册和发现。

1）服务注册

在服务治理框架中，通常都会构建一个注册中心，每个服务单元向注册中心登记自己提供的服务，包括服务的主机与端口号、服务版本号、通讯协议等一些附加信息。注册中心按照服务名分类组织服务清单，同时还**需要以心跳检测的方式去监测清单中的服务是否可用**，若不可用需要从服务清单中剔除，以达到排除故障服务的效果。

2）服务发现

在服务治理框架下，服务间的调用不再通过指定具体的实例地址来实现，而是通过服务名发起请求调用实现。服务调用方通过服务名从服务注册中心的服务清单中获取服务实例的列表清单，通过指定的负载均衡策略取出一个服务实例位置来进行服务调用。

- **Eureka介绍**

Spring Cloud Eureka是`Spring Cloud Netflix`微服务套件中的一部分，它基于`Netflix Eureka`做了二次封装。主要负责完成微服务架构中的服务治理功能。

`Spirng Cloud Eureka`使用`Netflix Eureka`来**实现服务注册与发现**。它既包含了服务端组件，也包含了客户端组件，并且服务端与客户端均采用java编写，所以`Eureka`主要适用于通过java实现的分布式系统，或是JVM兼容语言构建的系统。`Eureka`的服务端提供了较为完善的`REST API`，所以`Eureka`也支持将非java语言实现的服务纳入到`Eureka`服务治理体系中来，只需要其他语言平台自己实现`Eureka`的客户端程序。

1）Eureka服务端

`Eureka`服务端，即服务注册中心。它同其他服务注册中心一样，支持高可用配置。依托于强一致性提供良好的服务实例可用性，可以应对多种不同的故障场景。

**Eureka服务端支持集群模式部署**，当集群中有分片发生故障的时候，`Eureka`会自动转入自我保护模式。它允许在分片发生故障的时候继续提供服务的发现和注册，当故障分配恢复时，集群中的其他分片会把他们的状态再次同步回来。集群中的的不同服务注册中心通过异步模式互相复制各自的状态，这也意味着在给定的时间点每个实例关于所有服务的状态可能存在不一致的现象。

2）Eureka客户端

`Eureka`客户端，主要处理服务的注册和发现。客户端服务通过注册和参数配置的方式，嵌入在客户端应用程序的代码中。在应用程序启动时，`Eureka`客户端向服务注册中心注册自身提供的服务，**并周期性的发送心跳来更新它的服务租约**。同时，他也能从服务端查询当前注册的服务信息**并把它们缓存到本地并周期性地刷新**服务状态。

## 部署

#### 单机部署

- **服务器端**

1）pom文件添加依赖

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
  </dependency>
</dependencies>
```

2）启动类加注解

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {
	public static void main(String[] args) {
		SpringApplication.run(EurekaApplication.class, args);
  	}
}
```

3）application.xml配置

```yaml
spring:
  application:
    name: land-eureka
server:
  port: 8761
eureka:
  client:
    # 是否要注册到其他Eureka Server实例
    register-with-eureka: false
    # 是否要从其他Eureka Server实例获取数据
    fetch-registry: false
    service-url: 
      defaultZone: http://localhost:8761/eureka/
```

------

- **客户端**

1）添加依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

2）加注解

```java
@SpringBootApplication
public class ProviderUserApplication {
	public static void main(String[] args) {
   		SpringApplication.run(ProviderUserApplication.class, args);
  	}
}
```

**注意**：早期的版本（Dalston及更早版本）还需在启动类上添加注解`@EnableDiscoveryClient` 或`@EnableEurekaClient` ，从Edgware开始，该注解可省略。

3）添加配置

```yaml
server:
  port: 8001

spring:
  application:
    # 指定注册到eureka server上的服务名称
    name: land-user
eureka:
  client:
    service-url:
      # 指定eureka server地址
      defaultZone: http://localhost:8761/eureka/
  instance:
    # 是否注册IP到eureka server。如不指定或设为false，会注册主机名到eureka server
    prefer-ip-address: true
```

------

- **测试**

启动`Eureka`服务端，启动`land-user`服务，使用浏览器打开 http://localhost:8761/ 访问`eureka`管理页面，即可查看服务信息。

![0.png](https://i.loli.net/2021/03/18/JdD6XaFoQrHhBm8.png)

#### 集群部署

在生产环境中，通常需要配置3台及以上的服务注册中心来提高可用性，配置原理都一样，将注册中心分别指向其它的注册中心。这里介绍三台集群的配置情况，其实和双节点的注册中心类似，每台注册中心分别又指向其它两个节点即可，使用`application.yml`来配置。

三个配置文件，分别如下：

application-sever1.yml

```yaml
spring:
  application:
    name: land-eureka
  profiles: sever1
server:
  port: 8761
eureka:
  instance:
    hostname: sever1
  client:
    serviceUrl:
      defaultZone: http://sever2:8762/eureka/,http://sever3:8763/eureka/
```

application-sever2.yml

```yaml
spring:
  application:
    name: land-eureka2  #不能重复
  profiles: sever2
server:
  port: 8762
eureka:
  instance:
    hostname: sever2
  client:
    serviceUrl:
      defaultZone: http://sever1:8761/eureka/,http://sever3:8763/eureka/
```

application-sever3.yml

```yaml
spring:
  application:
    name: land-eureka3	#不能重复
  profiles: sever3
server:
  port: 8763
eureka:
  instance:
    hostname: sever3
  client:
    serviceUrl:
      defaultZone: http://sever1:8761/eureka/,http://sever2:8762/eureka/
```

修改本机`hosts`文件，添加`sever1`，`sever2`，`sever3`解析

```xml
127.0.0.1	sever1
127.0.0.1	sever2
127.0.0.1	sever3
```

然后在配置文件`application.yml`中`profiles`，`active`配置项分别指定`application-sever1.yml`，`application-sever2.yml`，`application-sever3.yml`启动服务即可。

```yaml
#使用的配置文件名
profiles:
	active: sever1  #分别指定sever1,sever2,sever3
```

## 原理

![1.png](https://i.loli.net/2021/03/18/E1GskMofPTBQlyx.png)

如图是`Eureka`集群的工作原理，包括：

1）Application Service，服务提供者；

2）Application Client，服务消费者；

3）Make Remote Call调用`RESTful API`；

4）us-east-1c、us-east-1d等都是`Availability Zone`，它们都属于`us-east-1`这个`region`。

由图可知，`Eureka`包含两个组件：`Eureka Server`和`Eureka Client`，它们的作用如下：

1）`Eureka Server`提供服务发现的能力，各个微服务启动时，会向`Eureka Server`注册自己的信息（例如IP、端口、微服务名称等），`Eureka Server`会存储这些信息；

2）`Eureka Client`是一个Java客户端，用于简化与`Eureka Server`的交互；

3）微服务启动后，会周期性（**默认30秒**）地向`Eureka Server`发送心跳以续约自己的“租期”；

4）如果`Eureka Server`在一定时间内没有接收到某个微服务实例的心跳，`Eureka Server`将会注销该服务实例（**默认90秒**）；

5）默认情况下，`Eureka Server`同时也是`Eureka Client`。多个`Eureka Server`实例，互相之间通过增量复制的方式，来实现服务注册表中数据的同步。`Eureka Server`默认保证在90秒内，`Eureka Server`集群内的所有实例中的数据达到一致；

6）`Eureka Client`会**缓存服务注册表中的信息**。这样，微服务无需每次请求都查询`Eureka Server`，从而降低了`Eureka Server`的压力；另外，即使`Eureka Server`所有节点都宕掉，服务消费者依然可以使用缓存中的信息找到服务提供者并完成调用。

`Eureka`通过心跳检查、客户端缓存等机制，提高了系统的可用性、灵活性以及可伸缩性。

