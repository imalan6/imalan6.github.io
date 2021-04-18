# RabbitMQ原理和使用方法总结

## 简介

### 消息队列

消息队列，即`MQ`(全称`Message Queue`)。消息队列是在消息的传输过程中保存消息的容器。它是典型的生产者—消费者模型。生产者不断向消息队列中生产消息，消费者不断的从队列中获取消息。消息队列提供一个异步通信机制，消息的发送者不必一直等到消息被成功处理才返回，而是可以立即返回。

消息中间件负责处理网络通信，如果网络连接不可用，消息被暂存于队列当中，当网络畅通的时候在将消息转发给订阅了队列的应用程序或服务处理。

消息的生产和消费都是异步的，消息队列只关心消息的发送和接收，没有业务逻辑的侵入，实现了生产者和消费者的解耦。

以电商系统为例，比如在用户下单时，订单服务需要调用库存服务进行商品扣库存操作。按照传统方式，下单要等到库存服务成功返回之后才能显示下单成功，如果这时网络出现故障导致库存服务一直没有返回，那用户将一直处于等待中，使得用户体验变得很差。在这种情况下，就可以在订单服务和库存服务之间使用消息队列进行解耦。订单服务将扣库存消息发送到消息队列中后直接返回，库存服务会自动去消息队列中获取消息处理，用户无需等待。这样既提高了并发量，又降低了服务之间的耦合度。

### JMS和AMQP

`JMS`（`Java MessageService`）是指`JMS API`。`JMS`是由Sun公司早期提出的消息标准，旨在为`Java`应用提供统一的消息操作，包括`create`、`send`、`receive`等。JMS已经成为`Java Enterprise Edition`的一部分。用户可以根据相应的接口实现`JMS`服务进行通信和相关操作。

`AMQP`（`Advanced Message Queue Protocol`）是一种协议，最早用于解决金融领域不同平台之间的消息传递交互问题。`AMQP`和`JMS`有本质的差别，`AMQP`不从`API`层进行限定，而是直接定义网络交换的数据格式。这使得实现了`AMQP`的消息发送者和消费者可以使用不同语言实现，只要采用相应的数据格式即可。

两者的区别如下：

- `JMS`定义了统一的接口，统一对消息的处理，而`AMQP`是通过规定协议来统一数据交互格式；

- `JMS`限定了必须使用`Java`语言；而`AMQP`只是协议，不规定实现方式，因此是跨语言的；

- `JMS`规定了两种消息模型，而`AMQP`的消息模型更加丰富。

常见的`MQ`产品如下：

- `ActiveMQ`：基于`JMS`规范实现，采用`Java`语言开发，是`Apache`下面的项目；

- `RabbitMQ`：基于`AMQP`协议实现，采用`Erlang`语言开发，稳定性好；

- `RocketMQ`：基于`JMS`规范实现，采用`Java`语言开发，阿里产品，目前交由`Apache`基金会；

- `Kafka`：由`Apache`软件基金会开发的开源流处理平台，采用`Scala`和`Java`开发，主要用于处理日志；

### RabbitMQ

`RabbitMQ`是一个开源的消息队列服务器。`RabbitMQ`基于`AMQP`协议实现，采用`Erlang`语言开发。`Erlang`语言在数据交互方面性能非常优秀，`RabbitMQ`具有很高的性能。`RabbitMQ`官方地址：http://www.rabbitmq.com

## 应用场景

- 异步处理，发送者把消息放入消息队列中可以直接返回，无需等到处理完成才返回。
- 应用解耦，消息发送者和消费者之间不必受对方的影响，只通过一个简单的容器来联系，可以更独立自主。
- 流量削峰，当请求量剧增时，发送者可以发请求发送到消息队列，当消息队列满了就拒绝响应，跳转到错误页面，这样就可以避免系统崩溃。
- 日志处理，将消息队列用在日志处理中，比如`Kafka`的应用，解决大量日志传输的问题。

## 工作原理

### 基本结构

`RabbitMQ`的基本结构如下图：

![2.png](https://i.loli.net/2021/04/06/yDa9gtF84CQrKvf.png)

- `Broker`：消息队列服务进程，包括两个部分：`Exchange`和`Queue`。

- `Exchange`：消息队列交换机，按一定的规则将消息路由转发到队列中，对消息进行过滤。

- `Queue`：消息队列，存储消息的队列，消息到达队列并转发给指定的消费者。多个消费者可以订阅同一个`Queue`，`Queue`中的消息会被平均分摊给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理。
- `Virtual Host`：虚拟主机，用于逻辑隔离。一个虚拟主机里面可以有若干个`Exchange`和`Queue`，同一个虚拟主机里面不能有相同名称的`Exchange`或`Queue`。
- `Connection`：连接，应用程序与`Broker`的网络连接，`TCP`连接。
- `Channel`：信道，消息读写等操作在信道中进行。客户端可以建立多个信道，每个信道代表一个会话任务。
- `Message`：消息，应用程序和服务器之间传送的数据，消息可以非常简单，也可以很复杂。
- `Binding`：绑定，交换器和消息队列之间的虚拟连接，绑定中可以包含一个或者多个`RoutingKey`。
- `RoutingKey`：路由键，生产者将消息发送给交换器的时候，会发送一个`RoutingKey`，用来指定路由规则，这样交换器就知道把消息发送到哪个队列。路由键通常为一个“.”分割的字符串，例如“`com.rabbitmq`”。

- `Producer`：消息生产者，即生产方客户端，生产方客户端将消息发送。

- `Consumer`：消息消费者，即消费方客户端，接收`MQ`转发的消息。

### 工作流程

生产者发送消息流程：

1）生产者和`Broker`建立`TCP`连接

2）生产者和`Broker`建立通道

3）生产者通过通道消息发送给`Broker`，由`Exchange`将消息进行转发

4）`Exchange`将消息转发到指定的`Queue`

消费者接收消息流程：

1）消费者和`Broker`建立`TCP`连接

2）消费者和`Broker`建立通道

3）消费者监听指定的`Queue`

4）当有消息到达`Queue`时`Broker`默认将消息推送给消费者

5）消费者接收到消息

6）`ack`回复

### PrefetchCount

当有多个消费者订阅同一个`Queue`时，`Queue`中的消息会被平摊给多个消费者处理。这时如果每个消费者处理消息的时间不同，就有可能导致某些消费者一直在处理，而另一些消费者很快就处理完手头工作并一直空闲的情况。遇到这种情况，可以通过设置`prefetchCount`来限制`Queue`每次发送给每个消费者的消息数，比如设置`prefetchCount=1`，则`Queue`每次给每个消费者发送一条消息，等消费者处理完这条消息后，`Queue`会再给该消费者发送一条消息。这样，处理速度快的消费者接受消息的频率就会快些，而处理速度慢的消费者接受消息的频率就会慢些，很好地均衡了多个消费者的工作量。

## Exchange类型

`RabbitMQ`常用的`Exchange Type`有：`direct`、`topic`、`fanout`、`headers`四种（`AMQP`规范里还提到两种`Exchange Type`，分别为`system`与自定义），下面分别进行介绍。

### Direct

`Direct`类型的`Exchange`路由规则很简单，它会把消息路由到`Routing Key`指定的队列中，也就是说`Routing Key`与`Binding Key`完全匹配的`Queue`中。

![3.png](https://i.loli.net/2021/04/06/MmgbRoVap6NSD5l.png)

如上图所示，如果用`routingKey=”orange”`发送消息到`Exchange`，则消息会路由到`Queue1`（`amqp.gen-S9b…`，这是由`RabbitMQ`自动生成的`Queue`名称），如果用`routingKey=”black”`或`routingKey=”green”`来发送消息，则消息会路由到`Queue2`。如果我们以其他`routingKey`发送消息，则消息不会路由到这两个`Queue`中。

### Topic

`Direct`类型的`Exchange`路由规则是完全匹配`binding key`与`routing key`，这种严格的匹配方式在很多情况下不能满足实际业务需求。`topic`类型的`Exchange`在匹配规则上进行了扩展，它与`direct`类型的`Exchange`相似，也是将消息路由到`binding key`与`routing key`相匹配的`Queue`中，但这里的匹配规则有些不同，它约定：

- `routing key`为一个句点号“. ”分隔的字符串（将被句点号“. ”分隔开的每一段独立的字符串称为一个单词），如“`com.rabbitmq`”、“`orange.abc`”、“`123.xyz.abc`”
- `binding key`与`routing key`一样也是句点号“. ”分隔的字符串
- `binding key`中可以存在两种特殊字符“\*”与“#”，用于做模糊匹配，其中“\*”用于匹配一个单词，“#”用于匹配多个单词（可以是零个）

![4.png](https://i.loli.net/2021/04/06/lZOsSjYbryRDaFT.png)

如上图所示：

- `routingKey`=”`abc.orange.rabbit`”的消息会同时路由到Q1与Q2

- `routingKey`=”`lazy.orange.abc`”的消息也会同时路由到Q1和Q2


- `routingKey`=”`lazy.xyz.abc`”的消息只会路由到Q2


- `routingKey`=”`lazy.xyz.rabbit`”的消息只会路由到Q2，但注意只会投递给Q2一次，虽然这个`routingKey`与Q2的两个`bindingKey`都匹配。


- `routingKey`=”`quick.xyz.abc`”、`routingKey`=”orange”、`routingKey`=”`abc.orange.xyz.rabbit`”的消息将会被丢弃，因为它们没有匹配任何`bindingKey`


### Fanout

`Fanout`类型的`Exchange`路由规则非常简单，它会把所有发送到该`Exchange`的消息路由到所有与它绑定的`Queue`中。

![5.png](https://i.loli.net/2021/04/06/w6RT5EAVlFPk2sd.png)

如上图所示，生产者（P）发送到`Exchange`（X）的所有消息都会路由到图中的两个`Queue`，并最终被C1与C2消费。

### Headers

`headers`类型的`Exchange`不依赖于`routing key`与`binding key`的匹配规则来路由消息，而是根据发送的消息内容中的`headers`属性进行匹配。

该类型的`Exchange`在实际中很少使用，不做过多介绍。

## 部署使用

### 部署

略，详见devops—docker部分

### Springboot集成RabbitMQ

添加配置：

```yaml
spring:
  application:
    name: land-service-producer
  #配置rabbitMq服务器
  rabbitmq:
    host: 127.0.0.1
    port: 5672
    username: admin
    password: admin
```

添加一个配置类，创建一个`exchange`和两个`queue`，并且让`exchange`和两个`queue`分别绑定起来，代码如下：

```java
@Configuration
public class RabbitMqConfig {
    public final static String QUEUE_NAME_1 = "land_queue1";
    public final static String QUEUE_NAME_2 = "land_queue2";
    public final static String EXCHANGE_NAME = "land_exchange";
    
    @Bean
    public Queue queue1() {
        // 创建一个queue 1
        return new Queue(RabbitMqConfig.QUEUE_NAME_1);
    }

    @Bean
    public Queue queue2() {
        // 创建一个queue 2
        return new Queue(RabbitMqConfig.QUEUE_NAME_2);
    }

    @Bean
    TopicExchange exchange() {
        // 创建一个topic exchange
        return new TopicExchange(EXCHANGE_NAME);
    }

    @Bean
    Binding bindingExchangeMessage() {
        // exchange和queue1绑定，并指定binding key
        return BindingBuilder.bind(queue1()).to(exchange()).with("topic.com.alan6.#");
    }

    @Bean
    Binding bindingExchangeMessages() {
        // exchange和queue2绑定，并指定binding key
        return BindingBuilder.bind(queue2()).to(exchange()).with("topic.message.*");
    }
}
```

发送者代码：

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = ProducerApplication.class)
public class RabbitMQTest {

    @Autowired
    private AmqpTemplate rabbitTemplate;

    @Test
    public void sendMessageTest() {
		// 测试消息
        String msg = "this is a test";
		// 路由key
        String routingKey = "topic.com.alan6.www";
		// 发送消息
        this.rabbitTemplate.convertAndSend(RabbitMqConfig.EXCHANGE_NAME, routingKey, msg);
    }
}
```

消费者代码：

```java
@Component
@RabbitListener(queues = "land_queue1")  //注解，消费land_queue1的消息
public class Consume {

    @RabbitHandler // 消费处理注解
    public void consumer(String msg){  // 参数为需要处理的消息
        System.out.println("consume msg：" + msg);
    }
}
```

### 测试

运行`rabbitmq`，启动`producer`测试代码，向`rabbitmq`发送消息：

![0.png](https://i.loli.net/2021/04/06/SspzkRPcOMbyQ29.png)

可以看见，`rabbitmq`已经接收到1个消息，正在等待消费，然后启动消费者应用：

![1.png](https://i.loli.net/2021/04/06/H2m5UJhBj1eNAfO.png)

`rabbitmq`中的消息已经被取走消费。

## Tracing记录消息

`RabbitMQ`的`Tracing`能跟踪`RabbitMQ`中消息的流入流出情况。`rabbitmq_tracing`插件会对流入流出的消息做封装，然后将封装后的消息存入相应的`trace`文件之中。

使用`rabbitmq-plugins enable rabbitmq_tracing`命令来启动`rabbitmq_tracing`插件。如果是使用`docker`部署的，先进入`docker`环境，再开启。

在`rabbitmq`的GUI管理界面 “Admin” 选项右侧原本只有 ”Users”、”Virtual Hosts”和 ”Policies“ 三项，在添加`rabbitmq_tracing`插件之后，会多出”Tracing”选项。

　![img](https://img2018.cnblogs.com/blog/1649754/201909/1649754-20190911094028149-1396092124.png)

- `Name`：自定义日志名称，建议标准点容易区分。

- `Format`：表示输出的消息日志格式，有`Text`和`JSON`两种，`Text`格式的日志方便人类阅读，`JSON`的方便程序解析。`Text`相比`JSON`格式占用空间稍大点。

- `JSON`格式的`payload`（消息体）默认会采用`Base64`进行编码。

- `Max payload bytes`：表示每条消息的最大限制，单位为B。比如设置了了此值为10，那么当有超过10B的消息经过`RabbitMQ`时会被截断。如消息“trace test payload.”会被截断成“trace test”。

- `Pattern`：用来设置匹配模式，如“#” 匹配所有消息流入流出的情况，即当有客户端生产消息或消费消息的时候，都会把相应的消息日志记录下来。“`publish.#`” 匹配所有消息流入的情况；“`deliver.#`” 匹配所有消息流出的情况；“`publish.exchange.b2b.gms.ass`”只匹配发送者(`Exchanges`)为`exchange.b2b.gms.ass`的所有消息流入的情况。

## 持久化

`RabbitMQ`默认是不持久`Exchange`、`Queue`、`Binding`以及队列中消息的，这意味着一旦`RabbitMQ`重启，所有已声明的队列，`Exchange`，`Binding`以及队列中的消息都会丢失。为了防止丢失，需要实现持久化。但持久化会对`RabbitMQ`的性能造成很大的影响，可能会下降10倍不止。所以，为了提高`rabbitmq`的性能，没有必要持久化的可以不用设置为持久化。

### Exchange 和 Queue 持久化

把`Exchange`和`Queue`的`durable`属性置为`true`，可以实现`Queue`和`Exchange`的持久化。但这里需要注意的是，只有`Exchange`和`Queues`的`durable`都为`true`时才能绑定，否则在绑定时，`RabbitMQ`会报错的。

### Message 持久化

消息的持久化需要在消息投递的时候设置`delivery mode`值为2。消息持久化必须同时要求`exchange`和`queue`也是持久化的。
持久化的代价就是性能损失,磁盘IO远远慢于RAM(使用SSD会显著提高消息持久化的性能) , 持久化会大大降低`RabbitMQ`每秒可处理的消息。两者的性能差距可能在10倍以上。

- exchange持久化，在声明时指定durable => 1

- queue持久化，在声明时指定durable => 1

- 消息持久化，在投递时指定delivery_mode => 2（1是非持久化）