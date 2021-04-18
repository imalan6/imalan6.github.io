# Netty零拷贝Zero-copy机制

## 传统数据拷贝方式

在传统数据拷贝方式中，如果遇到这样一个场景：需要从磁盘读取一个文件通过网络发送给客户端。服务端的实现步骤一般是使用如下函数：

```c
read(file, tmp_buf, len);
write(socket, tmp_buf, len);
```

从用户层面看，虽然只有两个操作步骤：从磁盘读取文件，再将文件写入到`socket`。但是在操作系统内部经历了一个较为复杂的过程，这个过程如下图所示：

![0.png](https://i.loli.net/2021/04/07/KEwIGpqV2ADmJr5.png)

从图中可以看出经历了4次数据拷贝过程：

1）首先，调用`read`方法，文件从`user`模式拷贝到了`kernel`模式，即用户模式—>内核模式的上下文切换，在内部发送`sys_read()`从文件中读取数据，存储到一个内核地址空间缓存区中。这一步，**数据从磁盘复制到内核缓冲区**。

2）之后CPU控制将`kernel`模式数据拷贝到`user`模式下，即内核模式—>用户模式的上下文切换，`read()`调用返回，数据被存储到用户地址空间的缓存区中。这一步，**数据从内核缓冲区复制到用户空间缓冲区**。

3）调用write时候，先将user模式下的内容拷贝到`kernel`模式下`socket`的`buffer`中，即用户模式—>内核模式，数据再次被放置在内核缓存区中，`send()`套接字调用。这一步，**数据从用户缓冲区复制到内核的socket缓冲区**。

4）最后将`kernel`模式下的`socket buffer`的数据拷贝到网卡设备中，`send()`套接字调用返回。这一步，**数据从socket缓冲区复制到协议引擎**（这里是网卡驱动）。

很明显，第2、3次数据拷贝是多余的，白白让数据在用户空间转了一圈，即浪费了时间，又降低了性能。

![3.png](https://i.loli.net/2021/04/07/g8iPqMIlKNy9R1e.jpg)

## 零拷贝机制

零拷贝，即所谓的`Zero-copy`，是指在操作数据时，不需要将数据从一个内存区域拷贝到另一个内存区域。因为少了一次或多次数据的拷贝，因此程序性能得到提升。如果操作的数据量大，这种性能提升效果是非常明显的。

在操作系统层面上的`Zero-copy`通常指避免在用户态(User-space)与内核态(Kernel-space)之间来回拷贝数据。例如，Linux提供的`mmap`系统调用，它可以将一段用户空间内存映射到内核空间，当映射成功后，用户对这段内存区域的修改可以直接映射到内核空间。同样地, 内核空间对这段区域的修改也可以直接映射到用户空间。这样，就不需要在用户态与内核态之间拷贝数据，提高了数据传输的效率。

从上面传统数据拷贝方式看，第2、3次数据拷贝根本就是多余的。应用程序只是起到缓存数据传回到套接字的作用而已，没有其他意义。

应用程序使用`zero-copy`请求`kernel`直接把`disk`的数据传输到`socket`中，而不需要通过应用程序中转。`zero-copy`大大提高了应用程序的性能，并且减少了`kernel`和`user`模式的上下文切换。

数据可以直接从`read buffer`读缓存区传输到套接字缓冲区，也就省去了将操作系统的`read buffer`拷贝到程序的`buffer`，以及从程序`buffer`拷贝到`socket buffer`的步骤，直接将`read buffer`拷贝到`socket buffer`。`JDK NIO`中的`transferTo()` 方法就能实现这个操作，它依赖于操作系统底层的`sendFile()`实现。

```java
public void transferTo(long position, long count, WritableByteChannel target);
```

而底层是调用的`sendFile()`方法：

```c
#include <sys/socket.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

![2.png](https://i.loli.net/2021/04/07/nyimaFuILMChZed.png)

使用了`zero-copy`技术后，整个过程如下：

1）`transferTo()`方法使得文件的内容直接拷贝到了一个`read buffer`（kernel buffer）中

2）然后数据从`kernel buffer`拷贝到`socket buffer`中

3）最后将`socket buffer`中的数据拷贝到协议引擎（网卡设备）中传输

这里没有了从内核态到用户态的过程，上下文切换从4次减少到2次，同时把数据拷贝的次数从4次降低到3次。

linux 2.1内核开始引入了`sendfile`函数，用于将文件通过`socket`传输。

```
sendfile(socket, file, len);
```

该函数通过一次调用完成了文件的传输。该函数通过一次系统调用完成了文件的传输，减少了原来`read/write`方式的模式切换。此外更是减少了数据的拷贝。

通过`sendfile`传送文件只需要一次系统调用，当调用`sendfile`时：

1）首先通过`DMA`将数据从磁盘读取到`kernel buffer`中

2）然后将`kernel buffer`数据拷贝到`socket buffer`中

3）最后将`socket buffer`中的数据拷贝到网卡设备中（protocol buffer）发送；

`sendfile`与`read/write`模式相比，少了一次`copy`。但是从上述过程中发现从`kernel buffer`中将数据`copy`到`socket buffer`也是没有必要的。

因此，Linux 2.4内核对`sendfile`做了改进，如图所示：

![4.png](https://i.loli.net/2021/04/07/dHUsjYBLKC2N8WG.png)

改进后的处理过程如下：

1）将文件拷贝到`kernel buffer`中；(`DMA`引擎将文件内容拷贝到内核缓存区)

2）向`socket buffer`中追加当前要发生的数据在`kernel buffer`中的位置和偏移量

3）根据`socket buffer`中的位置和偏移量直接将`kernel buffer`的数据拷贝到网卡设备（protocol engine）中

linux 2.1内核中的 “数据被拷贝到`socket buffer`”的动作，在Linux 2.4内核做了优化，取而代之的是只包含关于数据的位置和长度的信息的描述符被追加到了`socket buffer`缓冲区中。`DMA`引擎直接把数据从内核缓冲区传输到网卡设备（protocol engine），从而减少了最后一次拷贝。经过上述过程，数据只经过了2次copy就从磁盘传送出去了。这就是`Zero-Copy`(这里的零拷贝是针对`kernel`来讲的，数据在`kernel`模式下是`Zero-Copy`)。Java中的`TransferTo()`方法实现了`Zero-Copy`。

`Zero-Copy`技术的使用场景有很多，比如`Kafka`，`Netty`等，可以大大提升程序性能。

## Netty的零拷贝机制

而需要注意的是，Netty中的`Zero-copy`与上面提到的操作系统层面上的`Zero-copy`不完全一样，Netty的`Zero-coyp`应该是多维度的。包含三个层次：

#### 避免从内核空间—>用户空间

这个和上述的操作系统系统层面的零拷贝机制一样，`Netty`在这一层对零拷贝实现就是`FileRegion`类的`transferTo()`方法，可以不提供`buffer`完成整个文件的发送，不再需要开辟`buffer`在用户-内核空间循环读写。

#### 避免从JVM Heap—>C Heap

在JVM层面，每当程序需要执行一个I/O操作时，都需要将数据先从JVM管理的堆内存复制到使用C malloc()或类似函数分配的Heap内存中，这样才能够触发系统调用完成操作，这部分内存对于Java应用来说是堆外内存，但是从操作系统来看其实都属于进程的堆区，操作系统并不知道JVM的存在，都是普通的用户程序。这样的话，JVM在I/O时永远比使用native语言编写的程序多一次数据复制，这也是所有基于虚拟机的编程语言都无法避免的问题。

而Netty是在适当的位置直接使用堆外内存从而避免了数据从JVM Heap到C Heap的拷贝。

#### 避免在用户空间的多次拷贝

在实现应用层数据传输功能时，可能会遇到这样一个场景。假如有一份协议数据, 它由头部和消息体组成, 而头部和消息体是分别存放在两个ByteBuf中，分别是:

```java
ByteBuf header
ByteBuf body
```

在代码处理时，通常会将header和body合并为一个ByteBuf来操作，这样处理起来更方便。比如：

```java
ByteBuf allBuf = Unpooled.buffer(header.readableBytes() + body.readableBytes());
allBuf.writeBytes(header);
allBuf.writeBytes(body);
```

可以看出，将header和body拷贝到新的allBuf中，这无形中增加了两次额外的数据拷贝操作。

而`Netty`提供了`CompositeByteBuf`类，它可以将物理上分散的多个`ByteBuf`从逻辑上当成一个完整的`ByteBuf`来操作，这样就免去了重新分配空间再复制数据的开销。`Netty`的`Zero-copy`体现在如下几个方面:

- Netty提供了 `CompositeByteBuf` 类，可以将多个`ByteBuf`合并为一个逻辑上的`ByteBuf`，避免了各个`ByteBuf`之间的数据拷贝。
- 通过`wrap`操作，可以将byte[]数组、`ByteBuf`、`ByteBuffer`等包装成一个`Netty ByteBuf`对象，进而避免了拷贝操作。
- `ByteBuf`支持`slice`操作，可以将`ByteBuf`分解为多个共享同一个存储区域的`ByteBuf`，避免了内存的拷贝。
- 通过`FileRegion`包装的`FileChannel.tranferTo`实现文件传输，可以直接将文件缓冲区的数据发送到目标`Channel`，避免了传统通过循环`write`方式导致的内存拷贝问题。

