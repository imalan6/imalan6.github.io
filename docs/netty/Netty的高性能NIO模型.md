# Netty的高性能NIO模型

## BIO模型

#### BIO模型

BIO，即同步阻塞IO。服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理。我们熟悉的`Socket`编程就是BIO，每个请求对应一个线程去处理。一个`socket`对应一个处理线程，这个线程负责这个`Socket`连接的一系列数据传输操作，如果这个连接不做任何事情必然会造成不必要的线程开销。

由于操作系统允许的线程数量是有限的，多个`socket`申请与服务端建立连接时，服务端不能提供相应数量的处理线程，没有分配到处理线程的连接就会阻塞等待或被拒绝。如下图是BIO通信模型：

![3.png](https://i.loli.net/2021/04/08/r2akopiDjg7BYPI.png)

同步阻塞IO的改进模型：采用线程池创建`socket`线程，但也只是解决了频繁创建线程的问题，无法根本性提升性能。

![4.png](https://i.loli.net/2021/04/08/IoMqwhFUKQBNxsY.png)

BIO的问题：

- 每个请求都需要创建独立的线程，与对应的客户端进行数据`Read/Write`，业务处理。
- 当并发量较大时，需要创建足够多的线程处理连接，系统资源消耗很大。
- 连接建立后，如果当前线程暂时没有数据可读，线程就会阻塞在`Read`操作上，造成线程资源浪费。

#### BIO Demo

```java
public class SocketServer {
    public static void main(String[] args) {
        try {
            ServerSocket server = new ServerSocket(8888);
            System.out.println("服务器已经启动！");
            // 接收客户端发送的信息
            Socket socket = server.accept();

            InputStream is = socket.getInputStream();
            BufferedReader br = new BufferedReader(new InputStreamReader(is));

            String info = null;
            while ((info = br.readLine()) != null) {
                System.out.println(info);
            }

            // 向客户端写入信息
            OutputStream os = socket.getOutputStream();
            String str = "欢迎登陆到server服务器!";
            os.write(str.getBytes());

            // 关闭文件流
            os.close();
            br.close();
            is.close();
            socket.close();
            server.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## NIO模型

NIO，即同步非阻塞IO。同步是指线程不断轮询IO事件是否就绪，非阻塞是指线程在等待IO时，可以同时做其他任务。

NIO 有三大核心部分：`Channel`(通道)，`Buffer`(缓冲区)，`Selector`(选择器)，`Selector`代替了线程本身轮询IO事件，避免了线程阻塞，同时减少了不必要的线程开销。当IO事件就绪时，可以把数据写到Buffer，保证IO成功，而无需线程阻塞式地等待。

通俗理解：NIO可以做到用一个线程来处理多个操作。假设有10000个请求过来，根据实际情况，可以分配50或者100个线程来处理。不像BIO，必须分配10000个线程处理。

Java NIO全称java non-blocking IO，从JDK1.4开始，Java提供了一系列改进的输入/输出的新特性。NIO相关类都放在`java.nio`包及子包下，并且对原http://java.io包中的很多类进行改写。Java NIO的非阻塞模式，当一个线程从`channel`获取数据时，它仅能得到目前可用的数据，如果目前没有数据可读时，就什么也不操作，直到数据可以读取之前，该线程可以继续做其他事情，而不会保持线程阻塞。非阻塞写也是一样，一个线程请求写入一些数据到`channel`，不需要等它完全写入，这个线程同时可以去做别的事情。

#### NIO和BIO的区别

- BIO以流的方式处理数据,而NIO以块的方式处理数据,块I/O的效率比流I/O高很多
- BIO是阻塞的，NIO则是非阻塞的
- BIO基于字节流和字符流进行操作，而NIO基于`Channel`(通道)和`Buffer`(缓冲区)进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。`Selector`(选择器)用于监听多个通道的事件（比如：连接请求，数据到达等），因此使用单个线程就可以监听多个客户端通道

## Reactor模型

NIO基于`Reactor`，是典型的`Reactor`模型结构。`Reactor`模式是一种事件处理的模式，将多个输入同时交付给服务处理程序，然后对请求进行多路分解，并同步地分发到到对应的处理器中。

`Reactor`设计模式是`event-driven architecture`的一种实现方式，处理多个客户端并发的向服务端请求服务的场景。每种服务在服务端可能由多个方法组成。`Reactor`会解耦并发请求的服务并分发给对应的事件处理器来处理。

#### Reactor单线程模型

每个客户端发起连接请求都会交给`acceptor`，`acceptor`根据事件类型交给线程`handler`处理，但是由于在同一线程中，容易产生一个`handler`阻塞影响其他的情况。

![0.png](https://i.loli.net/2021/04/08/aQEqUxfNnG69LOv.png)

对于一些并发量较小的应用场景，可以使用`Reactor`单线程模型。但对于高负载、高并发的应用场景却不合适，主要原因如下：

- 一个NIO线程同时处理大量连接，性能上无法支撑，即便NIO线程的CPU负荷达到100%，也无法满足海量消息的编码、解码、读取和发送；

- 当NIO线程负载过重之后，处理速度将变慢，这会导致大量客户端连接超时，超时之后往往会进行重发，这更加重了NIO线程的负载，最终会导致大量消息积压和处理超时，成为系统的性能瓶颈；

- 一旦NIO线程出问题，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障。

#### Reactor多线程模型

在经典的`Reactor`单线程模式中，尽管一个线程可同时监控多个请求，但是所有读/写请求以及对新连接请求的处理都在同一个线程中处理，无法充分利用多CPU的优势，同时读/写操作也会阻塞对新连接请求的处理。因此可以引入多线程，并行处理多个读/写操作。

这里使用了单线程进行接收客户端的连接，采用了NIO的线程池用来处理客户端对应的IO操作，由于客户端连接较多，采用一个线程处理多个连接，如下图所示：

![1.png](https://i.loli.net/2021/04/08/GyE2KXQ71xukM9o.png)

`Reactor`多线程模型的特点：

- 有专门一个NIO线程—`Acceptor`线程用于监听服务端，接收客户端的TCP连接请求；

- 网络IO操作-读、写等由一个NIO线程池负责，它包含一个任务队列和多个可用的线程，这些NIO线程负责消息的读取、解码、编码和发送；

- 1个NIO线程可以同时处理多个连接，但是1个连接只对应1个NIO线程，防止发生并发操作问题。

在绝大多数场景下，`Reactor`多线程模型都可以满足性能需求；但是，在极个别特殊场景中，一个NIO线程负责监听和处理所有的客户端连接可能会存在性能问题。例如并发百万客户端连接，或者服务端需要对客户端握手进行安全认证，但是认证本身非常损耗性能。在这类场景下，单独一个`Acceptor`线程可能会存在性能不足问题，为了解决性能问题，产生了第三种`Reactor`线程模型—`Reactor`主从多线程模型。

#### Reactor主从多线程模型

比起第二种多线程模型，它将`Reactor`分成两部分，`mainReactor`负责监听并`accept`新连接，然后将建立的`socket`通过多路复用器（`Acceptor`）分派给`subReactor`。`subReactor`负责多路分离已连接的`socket`，读写网络数据，业务处理功能，交给`worker`线程池完成。通常，`subReactor`个数上可与CPU个数等同。

![2.png](https://i.loli.net/2021/04/08/HgQnJNf82MVlRsp.png)

`Reactor`主从多线程模型流程如下：

- 从主线程池中随机选择一个`Reactor`线程作为`Acceptor`线程，用于绑定监听端口，接收客户端连接；

- `Acceptor`线程接收客户端连接请求之后创建新的`SocketChannel`，将其注册到主线程池的其它`Reactor`线程上，由其负责接入认证、IP黑白名单过滤、握手等操作；

- 第2步完成之后，业务层的链路正式建立，将`SocketChannel`从主线程池的`Reactor`线程的多路复用器上摘除，重新注册到Sub线程池的线程上，用于处理I/O的读写操作。

## AIO模型

NIO是同步IO，程序需要IO操作时，必须进行IO操作后才能进行下一步操作。AIO是对NIO的改进，它基于`Proactor`模型。每个`socket`连接在事件分离器注册 IO完成事件和IO完成事件处理器。程序需要进行IO时，向分离器发出IO请求并把所用的`Buffer`区域告知分离器，分离器通知操作系统进行IO操作，操作系统自己不断尝试获取IO权限并进行IO操作（数据保存在`Buffer`区），操作完成后通知分离器。

AIO是发出IO请求后，由操作系统自己去获取IO权限并进行IO操作；NIO则是发出IO请求后，由线程不断尝试获取IO权限，获取到后通知应用程序自己进行IO操作。

同步/异步：数据如果尚未就绪，是否需要等待数据结果。

阻塞/非阻塞：进程/线程需要操作的数据如果尚未就绪，是否妨碍了当前进程/线程的后续操作。应用程序的调用是否立即返回。
