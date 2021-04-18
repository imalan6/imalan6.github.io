# Introduction

Alan Notes：Java核心、微服务、系统架构、数据库、设计模式、大数据等笔记整理。

<details> <summary>内容目录</summary> 
<font size=3>Java核心</font>
- <font size=3>**Java核心**</font>

  

  - 集合

    - [ArrayList/Vector](collections/ArrayList.md)

    - [LinkedList](collections/LinkedList.md)

    - [HashMap](collections/HashMap.md)

    - [HashSet](collections/HashSet.md)

    - [LinkedHashMap](collections/LinkedHashMap.md)

      

  - 并发

    - [Java多线程使用](concurrency/Java多线程使用.md)

    - [多线程的三大核心](thread/Threadcore.md)

    - [线程池的使用](concurrency/线程池的使用.md)

    - [Callable和Future配合](concurrency/Callable和Future配合.md)

    - [Java锁的分类和使用](concurrency/Java锁的分类和使用.md)

    - [volatile关键字](concurrency/volatile关键字.md)

    - [Atomic原子类](concurrency/Atomic原子类.md)

    - [CompareAndSet](concurrency/CompareAndSet.md)

    - [synchronized关键字原理](concurrency/synchronized关键字原理.md)

    - [ThreadLocal本地局部变量](concurrency/ThreadLocal本地局部变量.md)

    - [ReentrantLock实现原理 ](concurrency/ReentrantLock实现原理.md)

      

  - JVM

    - [JVM运行时内存结构](jvm/JVM运行时内存结构.md)

    - [JVM垃圾回收机制](jvm/JVM垃圾回收机制.md)

    - [JVM常见垃圾回收器](jvm/JVM常见垃圾回收器.md)

    - [JVM类加载机制](jvm/JVM类加载机制.md)

    - [JVM类加载器和双亲委派机制](jvm/JVM类加载器和双亲委派机制.md)

    - [JVM调优总结](jvm/JVM调优总结.md)

    - [对象引用关系](jvm/对象引用关系总结.md)

    - [生产环境系统运行缓慢问题排查](jvm/生产环境系统运行缓慢问题排查.md)

    - [生产环境CPU 100%解决思路](jvm/生产环境CPU100解决思路.md)

    - [Thread Dump日志分析案例](jvm/ThreadDump日志分析案例.md)

      

  - 其他

    - [反射机制](java_others/反射机制.md)
    - [动态代理](java_others/动态代理.md)
    - [JDK动态代理原理分析](java_others/JDK动态代理原理分析.md)
    - [过滤器和拦截器](java_others/过滤器和拦截器.md)
    - [自定义注解](java_others/自定义注解.md)
    - [Java回调机制](java_others/Java回调机制.md)
    - [深拷贝与浅拷贝](java_others/深拷贝与浅拷贝.md)
    - [Java泛型](java_others/Java泛型.md)

  

- <font size=3>**分布式/微服务**</font>

  

  - SpringBoot/Spring Cloud

    - [SpringCloud系列之Eureka](springcloud/SpringCloud系列之Eureka.md)

    - [SpringCloud系列之Zuul](springcloud/SpringCloud系列之zuul.md)

    - [SpringCloud系列之Feign](springcloud/SpringCloud系列之Feign.md)

    - [SpringCloud系列之Ribbon](springcloud/SpringCloud系列之Ribbon.md)

    - [SpringCloud系列之Hystrix](springcloud/SpringCloud系列之Hystrix.md)

    - [SpringCloud系列之Actuator](springcloud/SpringCloud系列之Actuator.md)

    - [SpringCloud系列之Admin](springcloud/SpringCloud系列之Admin.md)

    - [SpringCloud系列之Sleuth和Zipkin](springcloud/SpringCloud系列之Sleuth和Zipkin.md)

    - [SpringBoot集成Netty](springboot/SpringBoot集成Netty.md)

    - [SpringBoot配置Slf4j和Logback](springboot/SpringBoot配置Slf4j和Logback.md)

      

  - 缓存/MQ/负载均衡

    - [SpringBoot整合Redis及常用工具类](cache/SpringBoot整合Redis及常用工具类.md)

    - [缓存雪崩、穿透、击穿解决方案](cache/缓存雪崩穿透击穿解决方案.md)

    - [Redis持久化方案RDB和AOF](cache/Redis持久化方案RDB和AOF.md)

    - [Redis+Token机制实现接口幂等性](cache/Redis实现接口幂等性方案.md)

    - [RabbitMQ原理和使用方法总结](mq/RabbitMQ原理和使用方法总结.md)

    - [RabbitMQ消息确认机制](mq/RabbitMQ消息确认机制.md)

    - [RabbitMQ可靠消息投递](mq/RabbitMQ可靠消息投递.md)

    - [Netty的高性能NIO模型](netty/Netty的高性能NIO模型.md)

    - [Netty零拷贝Zero-Copy机制](netty/Netty零拷贝Zero-copy机制.md)

    - [Netty检查连接断开的几种方法](netty/Netty检查连接断开的几种方法.md)

    - [Nginx反向代理集群部署](nginx/Nginx反向代理集群部署.md)

      

  - 分布式组件

    - [基于Redis的分布式限流](distributed_component/基于Redis的分布式限流.md)

    - [基于Redis的分布式锁](distributed_component/基于Redis的分布式锁.md)

    - [分布式缓存设计](distributed_component/分布式缓存设计.md)

    - [分布式 ID 生成器](distributed_component/分布式ID生成器.md)

    

- **数据库**

  

  - Mysql

    - [一次慢查询sql导致的故障排查](mysql/一次慢查询sql导致的故障排查.md)

    - [mysql和redis的数据一致性问题](mysql/mysql和redis的数据一致性问题.md)

    

- **DevOps**

  

  - Docker

    - [安装docker和docker compose](docker/安装docker和dockercompose.md)
    - [docker部署redis](docker/docker部署redis.md)
    - [docker部署rabbitmq和rabbitmq集群](docker/docker部署rabbitmq和rabbitmq集群.md)
    - [docker部署nginx配置SSL证书实现https](docker/docker部署nginx配置SSL证书实现https.md)
    - [docker部署elk日志采集系统（tcp方式）](docker/docker部署elk日志采集系统（tcp方式）.md)
    - [docker部署elk日志采集系统（kafka方式）](docker/docker部署elk日志采集系统（kafka方式）.md)
    - [docker部署nexus私有仓库](docker/docker部署nexus私有仓库.md)
    - [Idea使用docker插件部署服务到远程服务器](docker/Idea使用docker插件部署服务到远程服务器.md)

</details>

------

# About Me

![3.png](https://i.loli.net/2021/03/15/JtovsLR5caYbTgu.png)

Alan，毕业于四川大学，计算机硕，十年程序猿老兵。主要从事 Java 开发，架构设计等。热爱技术，喜欢研究解决技术问题。

业余爱好：运动健身，看书，电影。

座右铭：认真做人，踏实做事。



------


# Contact Me

个人邮箱：361526055@qq.com


