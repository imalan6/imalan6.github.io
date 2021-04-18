# RabbitMQ消息确认机制

## 两种确认机制

`RabbitMQ`的消息确认有两种：

- 对生产端发送消息的确认。这种是用来确认生产者将消息发送给交换器，交换器传递给队列的过程中，消息是否成功投递。发送确认分为两步，一是确认是否到达交换器，二是确认是否到达队列。

- 对消费端消费消息的确认。这种是确认消费者是否成功消费了队列中的消息。

#### 对生产端发送消息的确认

`rabbitmq`对生产端发送消息的确认分为事务和实现`confirm`机制。不过一般不使用事务，性能消耗太大。

对生产端的`confirm`机制详见：[RabbitMQ可靠消息投递](mq/RabbitMQ可靠消息投递.md)

#### 消费端消费消息后的确认

为了保证消息能可靠到达消费端，`RabbitMQ`也提供了消费端的消息确认机制。消费者在声明队列时，可以指定`noAck`参数，当`noAck=false`时，`RabbitMQ`会等待消费者显式发回`ack`后才从内存(和磁盘，如果是持久化消息的话)中移去消息。否则，`RabbitMQ`会在队列中消息被消费后立即删除它。

采用消息确认机制后，只要令`noAck=false`，消费者就有足够的时间处理消息，不用担心处理消息过程中消费者进程挂掉后消息丢失的问题，因为`RabbitMQ`会一直持有消息直到消费者显式调用`basicAck`为止。

消费者消息的确认分为：

- `AcknowledgeMode.AUTO`：自动确认（默认）

- `AcknowledgeMode.MANUAL`：手动确认

- `AcknowledgeMode.NONE`：不确认

在spring-boot中，消费者手动确认配置方法：

```properties
spring.rabbitmq.listener.simple.acknowledge-mode = manual
```

## 消费者手动确认

#### 消费成功手动确认方法

`void basicAck(long deliveryTag, boolean multiple) throws IOException`；

- `deliveryTag`：该消息的`index`

- `multiple`：是否批量确认。`true`：将一次性`ack`所有小于`deliveryTag`的消息。

消费者成功处理消息后，手动调用`channel.basicAck(message.getMessageProperties().getDeliveryTag(), false)`方法对消息进行消费确认。

```java
try {
      channel.basicAck(message.getMessageProperties().getDeliveryTag(), false); // 手动确认消息
      System.out.println("投递消息确认成功，tag："+message.getMessageProperties().getDeliveryTag());
} catch (IOException e) {
      e.printStackTrace();
}
```

#### 消费失败手动确认方法

`void basicNack(long deliveryTag, boolean multiple, boolean requeue) throws IOException`;

- `deliveryTag`:该消息的`index`。

- `multiple`：是否批量. `true`：将一次性拒绝所有小于`deliveryTag`的消息。

- `requeue`：被拒绝的是否重新入队列。

`void basicReject(long deliveryTag, boolean requeue) throws IOException`;

- `deliveryTag`:该消息的`index`。

- `requeue`：被拒绝的是否重新入队列。

`channel.basicNack`方法与`channel.basicReject`方法区别在于`basicNack`可以批量拒绝多条消息，而`basicReject`一次只能拒绝一条消息。

## 消费者手动确认可能的问题

#### 消息无法ack

消费端在消费消息过程中出现异常，不能回复`ack`应答，消息将变成`unacked`状态，且一直处于队列中。如果积压的消息过多将会导致程序无法继续消费数据。

消费端服务重启，断开`rabbitmq`的连接后，`unacked`的消息状态会重新变为`ready`等待消费。但是如果不重启消费端服务，消息将一直驻留在`RabbitMQ`中。

所以，如果想不重启消费端的话，可以捕获异常，然后调用`basicNack`，让消息重新进入队列再次消费。

#### 无效消息循环重入队列

在上一个问题中，如果消费端捕获异常，并执行`basicNack`应答，将消息重新放入队列中，可能会出现另一个问题：

如果消息或者代码本身有`bug`，每次处理这个消息都会报异常，那消息将一直处于消费——>报异常——>重入队列——>继续消费——>报异常。。。的死循环过程。 

以上两个问题其实属于同一类问题，都需要我们确保代码在消费消息后，一定要通知`RabbitMQ`，不然消息将一直驻留在`RabbitMQ`中。如果消息成功消费，则调用`channel.basicAck`正常通知`RabbitMQ`；如果消费失败，则调用`channel.basicNack`或者`channel.basicReject`确认消费失败。

但防止死循环有两种处理办法：

1）根据处理过程中报的不同异常类型，选择消息要不要重入队列。

```java
enum Action {
  ACCEPT,  // 处理成功
  RETRY,   // 可以重试的错误
  REJECT,  // 无需重试的错误
}

Action action = Action.RETRY; 
try {
    // 如果成功完成则action=Action.ACCEPT
}
catch (Exception e) {
   // 根据异常种类决定是ACCEPT、RETRY还是 REJECT
}
finally {
  // 通过finally块来保证Ack/Nack会且只会执行一次
  if (action == Action.ACCEPT) {
    channel.basicAck(tag);
  } else if (action == Action.RETRY) {
     channel.basicNack(tag, false, true);
  } else {
     channel.basicNack(tag, false, false);
  }  
} 
```

2）将处理失败的消息放入另一个队列中，手动取出处理。