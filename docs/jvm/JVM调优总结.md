# JVM调优总结

## 性能调优

性能调优包含多个层次，包括架构调优、代码调优、`JVM`调优、数据库调优和操作系统调优等。架构调优和代码调优是`JVM`调优的基础，其中架构调优是对系统影响最大的。

## JVM调优目标

`JVM`调优的最终目的是让应用程序以最小的硬件承载更大的吞吐。`JVM`调优主要是针对垃圾收集器的收集性能优化，让应用程序占用更少的内存和延迟，但拥有更大的吞吐量。`JVM`内存系统级调优主要的目的是减少`GC`的频率和`Full GC`的次数。

`Full GC`：包括`Young`、`Tenured`和`Perm`。因为`Full GC`需要对整个堆进行回收，所以比较慢，因此应尽可能减少`Full GC`次数。

导致`Full GC`的原因：

- 年老代（Tenured）被写满：尽量让对象在新生代`GC`时就被回收，让对象在新生代多存活一段时间。另外，不要创建过大的对象及数组，避免其直接在老年代创建对象 。

- 永久代`Pemanet Generation`空间不足（Java1.8 以后没有永久代）：增大`Perm Gen`空间，避免太多静态对象 ， 控制好新生代和旧生代的比例

- `System.gc()`被显示调用：垃圾回收不要手动触发，尽量由 JVM 触发垃圾回收

## 何时需要JVM调优

遇到以下情况，就需要考虑`JVM`调优了：

- `Heap`内存（老年代）持续上涨达到设置的最大内存值；
- `FullGC`次数频繁；
- `GC`停顿时间过长（超过1秒）；
- 应用出现`OutOfMemory`等内存异常；
- 应用中有使用本地缓存且占用大量内存空间；
- 系统吞吐量与响应性能不高或下降。

## JVM调优量化目标

`JVM`调优的量化目标参考实例：

- Heap 内存使用率 <= 70%;
- Old generation 内存使用率<= 70%;
- `avgpause` <= 1秒;
- `Full gc`次数为0 或`avg pause interval`>= 24 小时 ;

注意：不同应用的 JVM 调优量化目标是不一样的，这里只是一种建议。

## JVM调优步骤

一般情况下，`JVM`调优可通过以下步骤进行：

- 分析`GC`日志及`dump`文件，判断是否需要优化，确定瓶颈问题点；
- 确定`JVM`调优量化目标；
- 确定`JVM`调优参数（根据历史`JVM`参数来调整）；
- 依次调优内存、延迟、吞吐量等指标；
- 对比观察调优前后的差异；
- 不断的分析和调整，直到找到合适的`JVM`参数配置；
- 找到最合适的参数，将这些参数应用到所有服务器，并进行后续跟踪。

以上操作步骤需要多次不断迭代完成。一般是从满足程序的内存使用需求开始，然后是时间延迟要求，最后是吞吐量要求，要基于这个步骤不断优化。

## JVM参数规范

`JVM`调优最主要的方法就是通过修改`JVM`参数达到调优的目的。

-XX 参数被称为不稳定参数，此类参数的设置很容易引起`JVM`性能上的差异，使`JVM`存在极大的不稳定性。如果此类参数设置合理将大大提高`JVM`的性能及稳定性。

不稳定参数语法规则包含以下内容。

布尔类型参数值：

- -XX:+ '+'表示启用该选项
- -XX:- '-'表示关闭该选项

数字类型参数值：

- -XX:=给选项设置一个数字类型值，可跟随单位，例如：'m'或'M'表示兆字节;'k'或'K'千字节;'g'或'G'千兆字节。32K与32768是相同大小的。

字符串类型参数值：

- -XX:=给选项设置一个字符串类型值，通常用于指定一个文件、路径或一系列命令列表。例如：`-XX:HeapDumpPath=./dump.core`

## JVM部分参数解析

比如以下参数示例：

```bash
-Xmx4g –Xms4g –Xmn1200m –Xss512k -XX:NewRatio=4 -XX:SurvivorRatio=8 -XX:PermSize=100m -XX:MaxPermSize=256m -XX:MaxTenuringThreshold=15
```

上面为 Java7 及以前版本的示例，在 Java8 中永久代的参数`-XX:PermSize`和`-XX:MaxPermSize`已经失效。

参数解析：

- -Xmx4g：堆内存最大值为4GB。
- -Xms4g：初始化堆内存大小为4GB。
- -Xmn1200m：设置年轻代大小为1200MB。增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。
- -Xss512k：设置每个线程的堆栈大小。JDK5.0 以后每个线程堆栈大小为1MB，以前每个线程堆栈大小为256K。应根据应用线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。
- -XX:NewRatio=4：设置年轻代（包括`Eden`和两个`Survivor`区）与年老代的比值（除去持久代）。设置为4，则年轻代与年老代所占比值为1 : 4，年轻代占整个堆栈的1/5
- -XX:SurvivorRatio=8：设置年轻代中`Eden`区与`Survivor`区的大小比值。设置为8，则两个`Survivor`区与一个`Eden`区的比值为2 : 8，一个`Survivor`区占整个年轻代的1/10
- -XX:PermSize=100m：初始化永久代大小为100MB。
- -XX:MaxPermSize=256m：设置持久代大小为256MB。
- -XX:MaxTenuringThreshold=15：设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过`Survivor`区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在`Survivor`区进行多次复制，这样可以增加对象在年轻代的存活时间，让对象仅可能在年轻代即被回收。

新生代、老生代、永久代的参数，如果不进行指定，虚拟机会自动选择合适的值，同时也会基于系统的开销自动调整。

可调优参数：

-Xms：初始化堆内存大小，默认为物理内存的1/64(小于1GB)。

-Xmx：堆内存最大值。默认（MaxHeapFreeRatio 参数可以调整）空余堆内存大于70%时，`JVM`会减少堆直到`-Xms`的最小限制。

-Xmn：新生代大小，包括`Eden`区与2个`Survivor`区。

-XX:SurvivorRatio=1：`Eden`区与一个`Survivor`区比值为1 : 1。

-XX:MaxDirectMemorySize=1G：直接内存。报`java.lang.OutOfMemoryError: Direct buffer memory`异常可以上调这个值。

-XX:+DisableExplicitGC：禁止运行期显式地调用`System.gc()`来触发`FulllGC`。

注意: Java RMI 的定时 GC 触发机制可通过配置`-Dsun.rmi.dgc.server.gcInterval=86400`来控制触发的时间。

-XX:CMSInitiatingOccupancyFraction=60：老年代内存回收阈值，默认值为68。

-XX:ConcGCThreads=4：CMS 垃圾回收器并行线程线，推荐值为 CPU 核心数。

-XX:ParallelGCThreads=8：新生代并行收集器的线程数。

-XX:MaxTenuringThreshold=10：设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过`Survivor`区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在`Survivor`区进行多次复制，这样可以增加对象在年轻代的存活时间，让对象仅可能在年轻代即被回收。

-XX:CMSFullGCsBeforeCompaction=4：指定进行多少次`FullGC`之后，进行`tenured`区 内存空间压缩。

-XX:CMSMaxAbortablePrecleanTime=500：当`abortable-preclean`预清理阶段执行达到这个时间时就会结束。

在设置的时候，如果关注性能开销的话，应尽量把永久代的初始值与最大值设置为同一值，因为永久代的大小调整需要进行`FullGC`才能实现。

## 内存优化示例

众所周知，由于`Full GC`的成本远远高于`Minor GC`，因此某些情况下需要尽可能将对象分配在年轻代，这在很多情况下是一个明智的选择。虽然在大部分情况下，`JVM`会尝试在`Eden`区分配对象，但是由于空间紧张等问题，很可能不得不将部分年轻对象提前向年老代压缩。因此，在`JVM`参数调优时可以为应用程序分配一个合理的年轻代空间，以最大限度避免新对象直接进入年老代的情况发生。如下所示代码尝试分配 4MB 内存空间，看一下它的内存使用情况。

```java
public class GCTest {
	public static void main(String[] args){
 		byte[] b1,b2,b3,b4;
 		b1=new byte[1024*1024];	//分配 1MB 堆空间，观察堆空间的使用情况
 		b2=new byte[1024*1024];
 		b3=new byte[1024*1024];
 		b4=new byte[1024*1024];
 	}
}
```

并使用`JVM`参数`-XX:+PrintGCDetails -Xmx20M -Xms20M`运行以上代码，输出结果如下：

```ASN.1
[GC (Allocation Failure) [PSYoungGen: 4673K->504K(6144K)] 4673K->3083K(19968K), 0.0023390 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 6144K, used 2717K [0x00000000ff980000, 0x0000000100000000, 0x0000000100000000)
  eden space 5632K, 39% used [0x00000000ff980000,0x00000000ffba94d8,0x00000000fff00000)
  from space 512K, 98% used [0x00000000fff00000,0x00000000fff7e010,0x00000000fff80000)
  to   space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
 ParOldGen       total 13824K, used 2579K [0x00000000fec00000, 0x00000000ff980000, 0x00000000ff980000)
  object space 13824K, 18% used [0x00000000fec00000,0x00000000fee84c90,0x00000000ff980000)
 Metaspace       used 3178K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 344K, capacity 388K, committed 512K, reserved 1048576K
```

以上结果所示的日志输出显示年轻代`Eden`的大小有 6MB 左右，在分配空间的时候触发了MinorGC，并将对象移到了老年代。其中：

[PSYoungGen: 4673K->504K(6144K)] 4673K->3083K(19968K), 0.0023390 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 

以上`GC`日志各要素含义如下：

- **[PSYoungGen: 4673K->504K(6144K)]** ：[日志类型：YoungGC前新生代内存占用 -> YoungGC后新生代内存占用（新生代总共大小）] ；

- **4673K->3083K(19968K), 0.0023390 secs**：YoungGC 前 JVM 堆内存占用 -> YoungGC 后 JVM 堆内存占用（JVM 堆总大小），YoungGC 耗时；

-  **[Times: user=0.00 sys=0.00, real=0.00 secs]**：[YoungGC 用户耗时，YoungGC 系统耗时，YoungGC 实际耗时]

现在分配足够大的年轻代空间，使用`JVM`参数`-XX:+PrintGCDetails -Xmx20M -Xms20M -Xmn10M`再次运行以上代码，输出结果如下：

```ASN.1
Heap
 PSYoungGen      total 9216K, used 7093K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 86% used [0x00000000ff600000,0x00000000ffced410,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
  to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
 ParOldGen       total 10240K, used 0K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 0% used [0x00000000fec00000,0x00000000fec00000,0x00000000ff600000)
 Metaspace       used 3206K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 349K, capacity 388K, committed 512K, reserved 1048576K
```

通过两次结果对比，可以发现通过设置一个较大的年轻代，可以将年轻对象保存在年轻代。一般来说，`Survivor`区的空间不够，或者占用量达到 50%时，就会使对象进入年老代 (不管它的年龄有多大)。

## 延迟优化示例

对延迟性优化，首先需要了解延迟性需求及可调优的指标。

- 应用程序可接受的平均停滞时间: 此时间与测量的时间比较
- `GC`持续时间进行比较，可接受的`MinorGC`频率
- `GC`的频率与可容忍的值进行比较。
- 最大停顿时间与最差情况下`FullGC`的持续时间进行比较。
- 可接受的最大停顿发生的频率：基本就是`FullGC`的频率。

其中，平均停滞时间和最大停顿时间，对用户体验最为重要。对于上面的指标，相关数据采集包括：`MinorGC`的持续时间、统计`MinorGC`的次数、`FullGC`的最差持续时间、最差情况下，`FullGC`的频率。

![0.png](https://i.loli.net/2021/03/13/k7yd8MPG2AKm1OE.png)

如上图，`MinorGC`的平均持续时间 0.069 秒，`MinorGC`的频率为 0.389 秒一次。

新生代空间越大，`MinorGC`的`GC`时间越长，频率越低。如果想减少其持续时长，就需要减少其空间大小。如果想减小其频率，就需要加大其空间大小。

这里以减少了新生代空间 10% 的大小，来减小延迟时间。在此过程中，应该保持老年代和持代的大小不变化。调优后的参数如下变化:

```ASN.1
java -Xms359m -Xmx359m -Xmn126m -XX:PermSize=5m -XX:MaxPermSize=5m
```

## 吞吐量调优

吞吐量调优主要是基于应用程序的吞吐量要求而来的，应用程序应该有一个综合的吞吐指标，这个指标基于整个应用的需求和测试而衍生出来。

评估当前吞吐量和目标差距是否巨大，如果在20%左右，可以修改参数，加大内存，再次从头调试，如果巨大就需要从整个应用层面来考虑，设计以及目标是否一致了，重新评估吞吐目标。

对于垃圾收集器来说，提升吞吐量的性能调优的目标就是尽可能避免或者很少发生`FullGC`或者`Stop-The-World`压缩式垃圾收集（`CMS`），因为这两种方式都会造成应用程序吞吐降低。尽量在`MinorGC`阶段回收更多的对象，避免对象提升过快到老年代。

## 开启GC日志

- 开启语句

```ASN.1
-XX:+PrintGCDetails  -XX:+PrintGCDateStamps  
-Xloggc:/var/log/gc-regionserver.log   
-XX:+UseGCLogFileRotation  
-XX:NumberOfGCLogFiles=10  -XX:GCLogFileSize=512k
```

- 参数详解

-XX:+PrintGCDetails
输出 GC 的详细日志

-XX:+PrintGCDateStamps
输出 GC 的日期戳

-Xloggc:/var/log/gc-regionserver.log
GC 日志输出的路径

-XX:+UseGCLogFileRotation
打开 GC 日志滚动记录功能

-XX:NumberOfGCLogFiles
设置滚动日志文件的个数，必须大于等于1
日志文件命名策略是，.0, .1, …, .n-1，其中n是该参数的值

-XX:GCLogFileSize
设置滚动日志文件的大小，必须大于8k
当前写日志文件大小超过该参数值时，日志将写入下一个文件

- 其他有用参数

-XX:+PrintGCApplicationStoppedTime
打印 GC 造成应用暂停的时间

-XX:+PrintHeapAtGC
在进行 GC 的前后打印出堆的信息

-XX:+PrintTenuringDistribution
在每次新生代`young GC`时，输出幸存区中对象的年龄分布

## GC日志分析工具

- GCeasy

`GCeasy`是一款在线的`GC`日志分析器，可以通过`GC`日志分析进行内存泄露检测、`GC`暂停原因分析、`JVM`配置建议优化等功能，而且是可以免费使用的。

网址：https://gceasy.io/

- GCViewer

借助`GCViewer`日志分析工具，可以非常直观地分析出`JVM`待调优点。可从以下几方面来分析：

Memory：分析`Totalheap`、`Tenuredheap`、`Youngheap`内存占用率及其他指标，理论上内存占用率越小越好；

Pause：分析`Gc pause`、`Fullgc pause`、`Total pause`三个大项中各指标，理论上`GC`次数越少越好，`GC`时长越小越好；

工具下载地址：https://github.com/chewiebug/GCViewer