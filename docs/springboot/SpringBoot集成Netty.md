# SpringBoot集成Netty

#### 添加依赖

```xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.36.Final</version>
</dependency>
```

#### 服务处理类

```java
@Slf4j
public class NettyServerHandler extends ChannelInboundHandlerAdapter {
    
    //连接建立
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        log.info("连接建立");
    }

    //收到消息
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        log.info("服务器收到消息: {}", msg.toString());
        ctx.write("返回消息");
        ctx.flush();
    }
    
    //链接关闭
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        log.info("关闭连接");
    }

    //发生异常
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}
```

#### 初始化类

```java
public class ServerChannelInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel socketChannel) throws Exception {
        //添加编解码
        socketChannel.pipeline().addLast("decoder", new StringDecoder(CharsetUtil.UTF_8));
        socketChannel.pipeline().addLast("encoder", new StringEncoder(CharsetUtil.UTF_8));
        socketChannel.pipeline().addLast(new NettyServerHandler());
    }
}
```

#### Netty Server启动类

```java
@component
@Slf4j
public class NettyServer {    
    @PostConstruct
    public void start() throws InterruptedException {
        //主线程组
        EventLoopGroup bossGroup = new NioEventLoopGroup(5);
        //工作线程组
        EventLoopGroup workGroup = new NioEventLoopGroup(20);
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup,workerGroup)
                .channel(NioServerSocketChannel.class)
                //服务端可连接队列数,对应TCP/IP协议listen函数中backlog参数
                .option(ChannelOption.SO_BACKLOG, 1024)
                //设置TCP长连接,一般如果两个小时内没有数据的通信时,TCP会自动发送一个活动探测数据报文
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .localAddress(inetSocketAddress)
                .childHandler(serverChannelInitializer);
        ChannelFuture future = bootstrap.bind().sync();
        if(future.isSuccess()) {
            log.info("启动netty Server");
        }
    }

    @PreDestroy
    public void destory() throws InterruptedException {
        bossGroup.shutdownGracefully().sync();
        workerGroup.shutdownGracefully().sync();
        log.info("关闭netty");
    }
}
```

#### SpringBoot启动类

```java
@SpringBootApplication
public class NettyApplication {
	public static void main(String[] args) {
		SpringApplication.run(NettyApplication.class, args);
	}
}
```