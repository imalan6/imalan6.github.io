# docker部署elk日志采集系统（kafka方式）

## logback+elk，tcp方式发送 

环境搭建参考上一篇：[docker部署elk日志采集系统（tcp方式）](docker/docker部署elk日志采集系统（tcp方式）.md)

`tcp`方式存在的问题：`tcp`方式在日志量比较大，并发量较高的情况下，可能导致日志丢失。可以考虑采用`kafka`保存日志消息，做一个流量削峰。

## logback+kafka+elk

**1、docker安装zookeeper+kafka**

拉镜像：

```shell
#docker pull wurstmeister/zookeeper
#docker pull wurstmeister/kafka
```

运行`zookeeper`：

```shell
#docker run -d --name zookeeper --restart always --publish 2181:2181 --volume /etc/localtime:/etc/localtime wurstmeister/zookeeper:latest
```

运行`kafka`：

```shell
#docker run -d --name kafka --restart always --publish 9092:9092 --link zookeeper --env KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
--env KAFKA_ADVERTISED_HOST_NAME=kafka所在宿主机的IP \
--env KAFKA_ADVERTISED_PORT=9092 \
--volume /etc/localtime:/etc/localtime \
wurstmeister/kafka:latest
```

**2、配置logback发送到kafka**

在服务端的`pom`文件添加依赖

```xml
<dependency>
      <groupId>com.github.danielwegener</groupId>
      <artifactId>logback-kafka-appender</artifactId>
</dependency>
```

在`logback-spring.xml`配置文件中添加`appender`

```xml
    <appender name="kafka" class="com.github.danielwegener.logback.kafka.KafkaAppender">
        <encoder class="com.github.danielwegener.logback.kafka.encoding.LayoutKafkaMessageEncoder">
            <layout class="net.logstash.logback.layout.LogstashLayout" >
                <includeContext>true</includeContext>
                <includeCallerData>true</includeCallerData>
                <customFields>{"system":"test"}</customFields>
                <fieldNames class="net.logstash.logback.fieldnames.ShortenedFieldNames"/>
            </layout>
            <charset>UTF-8</charset>
        </encoder>
        <!--kafka topic 需要与配置文件里面的topic一致 -->
        <topic>kafka_elk</topic>
        <keyingStrategy class="com.github.danielwegener.logback.kafka.keying.HostNameKeyingStrategy" />
        <deliveryStrategy class="com.github.danielwegener.logback.kafka.delivery.AsynchronousDeliveryStrategy" />
        <producerConfig>bootstrap.servers=192.168.33.128:9092</producerConfig>
    </appender>

    <!-- 日志输出级别 -->
    <root level="INFO">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="FILE"/>
        <appender-ref ref="kafka" />
    </root>
```

**3、配置logstash**

启动`elk`，进入容器：

```shell
#docker exec -it elk /bin/bash
```

进入`/etc/logstash/conf.d/`目录，创建配置文件`logstash.conf`，编辑内容，主要是`input`和`output`

```properties
input {
    kafka {
        bootstrap_servers => ["192.168.33.128:9092"]
        auto_offset_reset => "latest"
        consumer_threads => 5
        decorate_events => true
        group_id => "elk"
        topics => ["elk_kafka"]
        type => "bhy"
    }
}

output {
    stdout {}
    elasticsearch {
          hosts => ["192.168.33.128:9200"]
          index => "kafka-elk-%{+YYYY.MM.dd}"
    }
}
```

编辑`/etc/init.d/logstash`，修改

```properties
LS_USER=root 	//原来默认为logstash
LS_GROUP=root 	//原为默认为logstash
```

修改完成后退出，重启`elk`容器

```shell
#docker restart elk
```

**4、配置kibana**

配置方式和上一篇[docker部署elk日志采集系统（tcp方式）](docker/docker部署elk日志采集系统（tcp方式）.md)差不多，`Index Pattern`选择`logstash`中配置的 “`kafka-elk-日期`“ 。