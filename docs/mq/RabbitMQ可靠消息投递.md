# RabbitMQ可靠消息投递

生产端向`rabbitmq`发送消息时，由于网络等原因可能导致消息发送失败。所以，`rabbitmq`必须有机制确保消息能准确到达`rabbitmq`。如果不能到达，必须反馈给生产端进行重发。

`RabbitMQ`消息的可靠性投递主要两种实现：

- 通过实现消费的重试机制，通过`@Retryable`来实现重试，可以设置重试次数和重试频率

- 生产端实现消息可靠性投递

两种方法消费端都可能收到重复消息，要求消费端必须实现幂等性消费。

## 消息投递到Exchange的确认模式

`rabbitmq`的消息投递的过程为：

*producer ——> rabbitmq broker cluster ——> exchange ——> queue ——> consumer*

- 生产端发送消息到`rabbitmq broker`后，异步接受从`rabbitmq`返回的`ack`确认信息。

- 生产端收到返回的`ack`确认消息后，根据`ack`是`true`还是`false`，调用`confirmCallback`接口进行处理。

在`application.yml`中开启生产端`confirm`模式

```yaml
spring:
  rabbitmq:
    publisher-confirms: true
```

实现`ConfirmCallback`接口中的`confirm`方法，`ack`为`true`表示消息发送成功，`ack`为`false`表示消息发送失败

```java
@Component
@Slf4j
public class RabbitTemplateConfig implements ConfirmCallback{
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {        
        rabbitTemplate.setConfirmCallback(this);   // 指定 ConfirmCallback
    }
    
    @Override
    public void confirm(CorrelationData correlationData, boolean ack, String cause) {
       if (!ack) {
          //try to resend msg
       } else {
          //delete msg in db
       }
   }
}
```

注意：在`confirmCallback`回调接口中是没有消息数据的，所以即使消息发送失败，生产端也无法在这个回调接口中直接重发，`confirmCallback`只能起到一个通知的作用。

## 消息投递失败的重发机制

如果`rabbitmq`返回`ack`失败，生产端也无法确认消息是否真的发送成功，也会造成数据丢失。最好的办法是使用`rabbitmq`的事务机制，但是`rabbitmq`的事务机制效率极低，每秒钟处理的消息仅几百条，不适合并发量大的场景。 

另外一种实现思路：

1）生产端保存每次发送的消息，如果发送成功就删除消息；

2）如果发送失败就取出消息重新发送；

3）如果超时还没有收到`mq`返回的`ack`，同样取出消息重新发送。

这样就可以避免消息丢失的风险。以使用`redis`保存消息`msg`为例，具体实现方案为：

1）生产端在发送消息之前，生成`ack`唯一确认的`id`；

2）以`ackId`为键，消息为`value`，保存进`redis`缓存，设置超时时间；

3）`redis`实现超时触发接口，当`key`过期时，重发消息并再次执行第2步；

4）生产端实现`ConfirmCallback`接口；

5）`ConfirmCallback`接口触发时，若`ack`为`true`，则直接删除此次`ackId`对应的`msg`；若`ack`为`false`，则将该`ackId`对应的`msg`取出重发；

网上另外的实现方案：

不通过设置`redis`超时时间触发超时事件进行重发，而是取出消息放入一个`ackFailList`中，然后开启定时任务，扫描`ackFailList`，重发失败的`msg`。

网上的这套方案思路上和上一个方案差不多，是采用的额外的`List`来保存发送失败的消息，由于`List`保存在内存中，不具备持久化的功能，如果生产端程序异常退出将导致消息丢失，所以并不安全。可以考虑保存到数据库中。

## 消息未投递到queue的退回模式

生产端通过实现`ReturnCallback`接口，启动消息失败返回，消息路由不到队列时会触发该回调接口。

在`application.yml`中开启`return`模式

```yaml
spring:
  rabbitmq:
    publisher-returns: true
```

实现`ReturnCallback`接口，可以获取消息主体内容，实现消息重发

```java
@Component
@Slf4j
public class RabbitTemplateConfig implements ReturnCallback {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @PostConstruct
    public void init() {        
        //指定 ReturnCallback
        rabbitTemplate.setReturnCallback(this);   
    }

    @Override
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
        log.info("消息主体 message : {}", message);
        log.info("消息主体 message : {}", replyCode);
        log.info("描述：{}", replyText);
        log.info("消息使用的交换器 exchange : {}", exchange);
        log.info("消息使用的路由键 routing : {}", routingKey);
    }
}
```