# SpringCloud系列之Sleuth和Zipkin

## 简介

当采用微服务架构开发时，服务会按照不同的维度进行拆分，各业务之间通过接口相互调用。一个用户请求，可能需要很多微服务的相互调用才能完成。如果在业务调用链路上任何一个微服务出现问题或者网络超时，都可能导致功能失败。因此，我们需要一些可以帮助理解系统行为，并用于系统性能分析的工具，以便发生故障的时候，能够快速地定位和解决问题。随着业务越来越多，几乎每一个请求都会形成一个复杂的分布式服务调用链路，而对于微服务之间调用链路的分析会越来越复杂，类似下图所示：

![3.png](https://i.loli.net/2021/03/26/cZh8VebGIql5xNk.png)

谷歌在`Dapper`的论文中，提出了”链路追踪“的概念。链路追踪就是指一次任务的开始到结束，期间调用的所有系统及耗时（时间跨度）都可以完整记录下来。而`Spring Cloud Sleuth`就是`Spring Cloud`推出的分布式跟踪解决方案，并兼容支持了`Zipkin`（`Twitter`推出的分布式链路跟踪系统）和其他基于日志的追踪系统。通过Sleuth我们可以很清楚地掌握每一个请求经过了哪些服务，每个服务处理了多少时间，这让我们可以很方便地理清各个微服务间的调用关系。另外，`Sleuth`还提供如下功能：

- **耗时分析：**通过`Sleuth`可以很清楚地了解到每个采样请求的耗时，从而分析出哪些服务调用比较耗时;

- **可视化错误：**对于程序未捕捉的异常，可以通过集成`Zipkin`服务界面上看到;

- **链路优化：**对于调用比较频繁的服务，可以针对这些服务实施一些优化措施。

为了实现平台无关、厂商无关的分布式服务跟踪，`CNCF`发布了布式服务跟踪标准`Open Tracing`。目前，业界使用最广泛的分布式链路跟踪系统是`Twitter`的`Zipkin`。而国内各大企业自己使用的，主要有淘宝的 “鹰眼”，京东的 “`Hydra`”，大众点评的 “`CAT`”以及新浪的 “`Watchman`”等。

## Sleuth

链路跟踪的跟踪单元是从用户发起请求(`Request`)到系统的边界开始，然后到系统返回响应(`Response`)为止的过程，整个过程称为一个`Trace`。

每个`Trace`中可能会调用多个服务，在每次调用服务时，添加一个调用记录，用来记录每次调用消耗的时间等信息，这个调用记录称为一个Span。

若干个有序的`Span`组成了一个`Trace`。在系统向外界提供服务时，会不断地有请求和响应发生，也就会不断地生成`Trace`，把这些带有`Span`的`Trace`统计出来，就构成了一幅完成的系统调用链路拓扑图。然后再附上`Span`中的响应时间，请求成功与否等信息，就可以在出现问题时，很方便地找到异常服务。

![0.png](https://i.loli.net/2021/03/26/zZni2k6VKcAwRlu.png)

一个`Trace`是一次完整的调用链路，内部包含多个`Span`。`Trace`和`Span`是一对多的关系，而`Span`与`Span`之间存在父子关系，一系列的`Span`组成了树状结构。

比如：用户请求调用服务 C 、服务 D 、服务 E，其中服务 C 就是一个 `Span`，如果在服务 C 中另起一个线程调用了服务 D，那么服务 D 就是服务 C 的子 `Span`，如果在服务 D 中又另起一个线程调用了服务 E，那么服务 E 就是服务 D 的子 `Span`，而这个 C -> D -> E 构成的调用链路就是一条`Trace`。将链路调用数据汇总，类似如下图所示：

![1.png](https://i.loli.net/2021/03/26/TRzrv4ixj5uyE8N.png)

## Zipkin

`Zipkin`是`Twitter`的一个开源项目，它基于`Google Dapper`实现，它致力于收集服务的定时数据，包括数据的收集、存储、查找和显示。可以使用它来收集各个服务器上请求链路的跟踪数据，并通过它提供的`REST API`接口查询跟踪数据以实现对分布式系统的监控程序，从而及时地解决微服务系统中的性能问题，`Zipkin`还提供了可视化的`UI`组件帮助我们直观地搜索跟踪信息和分析请求链路细节。

![2.png](https://i.loli.net/2021/03/26/Jl6FB9tj4vMVX8b.png)

`Zipkin`主要包括 4 个核心组件：

- **Collector：**收集器，主要用于处理从外部系统发送过来的跟踪信息，将这些信息转换为`Zipkin`内部处理的`Span`格式，以支持存储、分析、展示等功能。
- **Storage：**存储组件，主要对处理收集器接收到的跟踪信息进行存储，默认存储在内存中。可以修改此存储策略，使用其他存储，比如`Mysql`。
- **RESTful API：**`API`接口，用来提供外部访问接口。比如给客户端展示跟踪信息，或是外接系统访问监控数据等。
- **Web UI：**`UI`组件，基于`API`组件实现的上层`UI`显示，可以很方便地查询和分析跟踪信息。

`Zipkin`分为服务端和客户端两部分。客户端直接部署在微服务上，一旦发生服务间的调用，会被`Sleuth`监听到，然后生成`Trace`和`Span`信息并发送到`Zipkin`服务端。发送的方式有两种：一种是采用`Http`报文的方式，另一种是采用消息队列发送，比如`RabbitMQ`。

## 部署

#### 1.服务端

在使用`Spring Boot 2.x`版本后，官方就不推荐自行定制编译`Zipkin`，而是直接提供了编译好的`jar`包供使用。使用如下命令安装运行：

```bash
curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar
```

如果使用Docker的话，命令如下：

```bash
#docker pull openzipkin/zipkin
#docker run -d -p 9411:9411 openzipkin/zipkin
```

启动`Zipkin`服务端后，访问 http://localhost:9411/zipkin/ ，界面显示如下：

![4.png](https://i.loli.net/2021/03/26/IcU5zksnBlNt8Kd.png)

#### 2.客户端

1）添加依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

2）配置

```yaml
spring:
  sleuth:
    web:
      client:
        enabled: true
    sampler:
      probability: 1.0 #将采样比例设置为1.0，也就是全部都需要。默认是0.1
      sender:
        type: web #数据传输方式，web 表示以 HTTP 报文的形式向服务端发送数据
  zipkin:
    base-url: http://192.168.0.6:9411/ #zipkin服务器的地址
```

`Spring Cloud Sleuth`有一个`Sampler`策略，用来控制数据采样算法。采样器不会阻碍`span`相关`id`的产生，但是会对导出以及附加事件标签的相关操作产生影响。 `Sleuth`默认采样算法是`Reservoir sampling`，默认的采样比例为 0.1(即 10%)。可以通过`spring.sleuth.sampler.percentage`设置，值介于0.0到1.0之间，1.0表示全部采集。

#### 3.测试

模拟用户发起服务请求，然后查看打开`Zipkin UI`界面，显示链路请求数据如下：

![5.png](https://i.loli.net/2021/03/26/1AKGOR6P7BzxpMX.png)





