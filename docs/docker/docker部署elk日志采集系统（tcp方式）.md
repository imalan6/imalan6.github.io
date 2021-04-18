# docker部署elk日志采集系统（tcp方式）

## elk概念

`ELK`是`Elasticsearch`、`Logstash`、`Kibana`的简称，这三者是核心套件，但并非全部。

**Elasticsearch**：实时全文搜索和分析引擎，提供搜集、分析、存储数据三大功能。`Elasticsearch`是一套开放`REST`和`JAVA API`等结构提供高效搜索功能，可扩展的分布式系统。它构建于`Apache Lucene`搜索引擎库之上。

**Logstash**：用来搜集、分析、过滤日志的工具。它支持几乎任何类型的日志，包括系统日志、错误日志和自定义应用程序日志。它可以从许多来源接收日志，这些来源包括`syslog`、消息传递（例如`RabbitMQ`）和`JMX`，它能够以多种方式输出数据，包括电子邮件、`websockets`和`Elasticsearch`。

**Kibana**：基于`Web`的可视化图形界面，用于搜索、分析和可视化存储在`Elasticsearch`指标中的日志数据。它利用`Elasticsearch`的`REST`接口来检索数据，不仅允许用户创建他们自己的数据的定制仪表板视图，还允许他们以特殊的方式查询和过滤数据。

## docker安装elk

**1、拉镜像，运行** 

```shell
#docker pull sebp/elk
#docker run -d -it --name elk --restart always -p 5601:5601 -p 9200:9200 -p 5044:5044 -e ES_MIN_MEM=128m -e ES_MAX_MEM=2048m sebp/elk
```

5601 - Kibana web接口

9200 - Elasticsearch JSON接口

5044 - Logstash日志接收接口

`logstash`有多种接受数据的方式，这里使用`logback`直接通过`tcp`的方式发送日志到`logstash`，还可以使用`redis`作为消息队列对日志数据做一个中转。

**2、elasticsearch报错解决**

`elasticsearch`启动时可能遇到错误，报`elasticsearch`用户拥有的内存权限太小，至少需要262144。

解决方法：

```shell
sysctl -w vm.max_map_count=262144
```

查看结果：

```shell
sysctl -a|grep vm.max_map_count，显示 vm.max_map_count = 262144
```

上述方法修改之后，如果重启虚拟机将失效。可以在`/etc/sysctl.conf`文件最后添加一行`vm.max_map_count=262144`，即可永久修改。

## 配置elk采集日志（tcp方式）

**1、配置tcp方式发送日志**

在服务的pom.xml中添加依赖

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
</dependency>
```

在`logback-spring.xml`中添加 `appender`，发送日志到`logstash`

```xml
<springProperty scope="context" name="springAppName" source="spring.application.name"/>    

<appender name="logstash" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>192.168.0.6:5044</destination>
        <!-- <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder" /> -->
        <!-- 日志输出编码 -->
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <pattern>
                    <pattern>
                        {
                        　　"severity": "%level",
                        　　"service": "${springAppName:-}",
                        　　"trace": "%X{X-B3-TraceId:-}",
                        　　"span": "%X{X-B3-SpanId:-}",
                       　　 "exportable": "%X{X-Span-Export:-}",
                        　　"pid": "${PID:-}",
                        　　"thread": "%thread",
                        　　"class": "%logger{40}",
                        　　"rest": "%message"
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
    </appender>

    <!-- 日志输出级别 -->
    <root level="INFO">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="FILE"/>
        <appende-ref ref="logstash" />
    </root>
```

**2、配置logstash发送日志到 elasticsearch**

由于`sebp/elk`中`logstash`的`input`默认采用`filebeat`，这里采用`tcp`方式，所以首先需要进入`elk`容器中修改`input`的方式为`tcp`。`logstash`默认会使用 `etc/logstash/conf.d/`中的配置文件。

启动elk，进入容器：

```shell
#docker exec -it elk /bin/bash
```

进入`/etc/logstash/conf.d/`配置目录，修改`02-beats-input.conf`配置文件如下：

```properties
input {
    tcp {
        port => 5044
        codec => json_lines
    }
}

output {
    elasticsearch {
    hosts => ["localhost:9200"]
    }
}
```

修改完成后退出，重启elk容器

```shell
#docker restart elk
```

**3、配置kibana**

启动服务，日志就被发送到`logstash`中了，访问`localhost:5601`进入`kibana`界面。

配置`index pattern`，点击`Management ——> Index Patterns ——> Create index pattern` 

![0.png](https://i.loli.net/2021/04/17/8Ro59mUTBjh3A46.png)

 下面列出了两个可选的`index pattern`，输入`logstash*`，下一步。选择时间`@timestamp`，这样数据展示会以时间排序。

![1.png](https://i.loli.net/2021/04/17/HMf36WVpOSC2nwk.png)

最后，在`kabina`首页查看采集到的日志

![2.png](https://i.loli.net/2021/04/17/Vu3rkM81lYv2hPf.png)

 