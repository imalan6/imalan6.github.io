# Netty检查连接断开的几种方法

最近项目中需要判定客户端是否还在线，需要用到心跳检测机制。这里总结一下。

## 心跳检测机制

网络中接收和发送数据都是通过操作系统的socket实现的。但是如果套接字已经断开，那发送和接收数据就会出问题。

但如何判断套接字是否断开了呢？这就需要建立一种机制，能够检测通信对方是否还存活。如果已经断开，就要释放资源。这种机制通常采用心跳检测实现。

所谓的“心跳”就是定时发送一个自定义的结构体（心跳包或心跳帧），让对方知道自己“在线”，以确保链接的有效性。心跳检测规定定时发送心跳检测数据包，接收方接心跳包后回复，否则认为连接断开。

## Netty心跳检测方式

#### pipeline加入IdleStateHandler

Netty提供了心跳检测类`IdleStateHandler`，它有三个参数，分别是读超时时间、写超时时间、读写超时时间。

- `readerIdleTime`：读超时时间

- `writerIdleTime`：写超时时间

- `allIdleTime`：所有类型超时时间

这里最重要是的`readerIdleTime`，当设置了`readerIdleTime`以后，服务端`server`会每隔`readerIdleTime`时间去检查一次`channelRead`方法被调用的情况，如果在`readerIdleTime`时间内该`channel`上的`channelRead()`方法没有被触发，就会调用`userEventTriggered`方法。

```java
//读超时时间设置为10s，0表示不监控
ch.pipeline().addLast(new IdleStateHandler(10, 0, 0, TimeUnit.SECONDS));

//加入处理事件
ch.pipeline().addLast(new ServerHeartBeat());
```

#### 重写userEventTriggered方法

重写`ChannelInboundHandlerAdapter`处理类的`userEventTriggered`方法，在方法中处理`idleEvent.state() == IdleState.READER_IDLE`情况。

```java
public class ServerHeartBeat extends ChannelInboundHandlerAdapter {

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) { //超时事件
            IdleStateEvent idleEvent = (IdleStateEvent) evt;
            if (idleEvent.state() == IdleState.READER_IDLE) { //读
                ctx.channel().close();
            } else if (idleEvent.state() == IdleState.WRITER_IDLE) { //写

            } else if (idleEvent.state() == IdleState.ALL_IDLE) { //全部

            }
        }
        super.userEventTriggered(ctx, evt);
    }
}
```

## TCP连接心跳检测方式

#### TCP心跳机制

上面采用的是`Netty`的心跳检测机制，属于应用层自定义的心跳检测方式。

应用层实现心跳机制好处：

- 灵活，应用层心跳可以自由定义，可实现各时间间隔（秒级、毫秒级）的检测，包里还可以携带额外的信息。

- 通用，应用层的心跳不依赖于底层协议。如果后期想把TCP改为UDP，协议层不提供心跳机制了，但应用层的心跳依旧是通用的。

应用层实现心跳机制坏处：

- 增加开发量，由于使用特定的网络框架，还可能很增加代码结构的复杂度。

- 额外增加了网络通信数据包，流量消耗更大。

TCP协议本身也实现了心跳检测机制。所以，也可以使用TCP的心跳机制检测连接是否断开。

如果设置了心跳，TCP会在一定时间（比如设置的是3秒钟）内发送设置的次数的心跳（比如2次），并且该心跳信息不会影响自己定义的协议。

#### TCP心跳设置

TCP协议自带的保活功能，使用起来很简单。`Linux`环境下，修改`/etc/sysctl.conf`文件，添加以下内容：

```bash
#表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是7200秒（2小时），改为5秒钟。
net.ipv4.tcp_keepalive_time = 5

#如果对方不予应答，探测包的发送次数
net.ipv4.tcp_keepalive_probes = 5

#探测消息发送的频率
net.ipv4.tcp_keepalive_intvl = 1
```

这个是修改的`Linux TCP`发送探测包配置的。修改完成后输入以下命令生效

```bash
/sbin/sysctl -p
/sbin/sysctl -w net.ipv4.route.flush=1
```

还可以使用echo的方式修改，命令如下：

```bash
#echo 5 > /proc/sys/net/ipv4/tcp_keepalive_time
#echo 5 > /proc/sys/net/ipv4/tcp_keepalive_probes
#echo 1 > /proc/sys/net/ipv4/tcp_keepalive_intvl 
```

修改后查看下参数，验证设置是否已经生效。

```bash
#cat /proc/sys/net/ipv4/tcp_keepalive_time  
#cat /proc/sys/net/ipv4/tcp_keepalive_probes  
#cat /proc/sys/net/ipv4/tcp_keepalive_intvl 
```

这样设置完成以后，每次客户端断开连接后5秒后会执行以下方法。

#### 使用Netty，重写handlerRemoved方法

```java
public class TcpServerHandler extends SimpleChannelInboundHandler<ByteBuf>{

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throw Exception{

        //执行客户端断开连接后的业务操作
    }
}
```

这里的时间设置不一定适合各业务场景，需要根据具体的业务设置合适的时间间隔。有时候网络不稳定，一个探测包没有探测到，系统就自动断开连接了。

## 使用redis保存连接

服务端收到客户端发送的心跳数据包后，使用`redis`保存，同时设置超时时间，一旦超时，触发超时事件，表示客户端连接出问题了。

比如：心跳时间为60s，那么服务器端收到客户端心跳数据包后，保存到`redis`，并设置超时事件为70s（根据实际情况，需要大于心跳时间），一旦超时触发超时事件，进行连接断开逻辑处理。