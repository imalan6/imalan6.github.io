# SpringCloud系列之Admin

## 简介

`Spring Boot Admin`主要用于管理和监控`SpringBoot`应用程序。被监控的应用程序被看作是一个客户端点，以`http`方式或者使用注册中心直接注册到`Spring Boot Admin Server`上。注册成功后，可以通过`Spring Boot Admin`提供的`UI`界面，可以轻松监控查看`Actuator`端点的监控信息。监控信息挺多，列举部分常见功能如下：

- 显示健康状况
- 显示详细信息，例如
  - `JVM`和内存指标
  - `micrometer.io`指标
  - 数据源指标
  - 缓存指标
- 查看`JVM`系统和环境属性
- 查看`Spring Boot`配置属性
- 轻松的日志级管理
- 与`JMX-beans`交互
- 查看线程转储
- 查看`http`跟踪
- 下载`heapdump`

## 单服务监控

- **Server端配置**

1）添加`pom`依赖

```xml
<dependencies>
  <dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
    <version>2.1.6</version>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
</dependencies>
```

2）启动类添加注解`@EnableAdminServer`

```java
@SpringBootApplication
@EnableAdminServer
public class LandAdminApplication {

    public static void main(String[] args) {
        SpringApplication.run(LandAdminApplication.class, args);
    }

}
```

- **客户端配置**

1）添加`pom`依赖

```xml
<dependencies>
    <dependency>
      <groupId>de.codecentric</groupId>
      <artifactId>spring-boot-admin-starter-client</artifactId>
      <version>2.1.6</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

2）添加配置

```
spring:
  boot:
    admin:
      client:
        url: http://localhost:8000  #Admin Server地址 
        
#actuator启用监控
management:
  endpoints:
    web:
      exposure:
        include: "*"  #开放所有端点health，info，metrics，通过actuator/端点名，就可以获取对应的信息。如果不配置，默认只打开health和info端点
```

需要配置`admin server`的地址，还需要打开`actuator`端点。

- **测试**

启动`Admin`服务和需要监控的微服务，访问 http://127.0.0.1:8000/ ，显示如下：

![0.png](https://i.loli.net/2021/03/25/mypHBKzeMuF9Wwr.png)

点击查看服务详细信息如下：

![1.png](https://i.loli.net/2021/03/25/yc1L3TR8JahBg7k.png)

`Spring Boot Admin`以图形化方式展示了被监控服务的各项信息，这些信息大多都来自于`Spring Boot Actuator`提供的接口。

## 微服务监控

如果只是单个或少量的`Spring Boot`应用，采用上面直接在端点完成配置的方式就可以了。但如果是微服务系统，且后端有很多微服务的话，采用上面的配置方式就显得过于复杂和笨拙。这时可以借助注册中心`Eureka`来完成，让`Spring Boot Admin`自动从注册中心抓取应用的相关信息。

如果使用`Spring Cloud`的服务发现功能完成监控抓取，服务端点就不需要再单独添加`Admin Client`依赖，只需要`Admin Serve`r就可以了，其它内容会自动进行配置。

1）客户端和`Admin`服务端都完成`Eureka`配置，详见 [SpringCloud系列之Eureka](springcloud/SpringCloud系列之Eureka.md) 。

2）启动各个服务，访问 http://127.0.0.1:8000/ ，显示如下：

![2.png](https://i.loli.net/2021/03/25/gJxaQ7lyEU9kDZY.png)





























