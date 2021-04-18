# 基于Redis的分布式锁

## 简介

分布式锁，或者称为“全局锁”，在分布式环境中，保证锁只能被一个对象获取，经常出现在“避免数据重复处理”、“接口幂等”等应用场景。

分布式锁一般有三种实现方式：

- 基于数据库的分布式锁

- 基于`Redis`的分布式锁

- 基于`ZooKeeper`的分布式锁

本篇主要研究基于`Redis`实现分布式锁。虽然网上已经有各种介绍`Redis`分布式锁实现的博客，然而却有着各种各样的问题，这里做个研究探讨。

## 可靠性

首先，为了确保分布式锁可用，至少要确保锁的实现同时满足以下四个条件：

1. 互斥性。在任意时刻，只有一个客户端能持有锁。
2. 不会发生死锁。即使有一个客户端在持有锁的期间崩溃而没有主动解锁，也能保证后续其他客户端能加锁。
3. 容错性。只要大部分的`Redis`节点正常运行，客户端就可以加锁和解锁。
4. 高性能。由于分布式系统，难免出现高并发环境，因此分布式锁不能成为性能瓶颈。

## Jedis实现

添加依赖：

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
```

### 加锁操作

##### 正确示例

```java
public class RedisTool {

    private static final String LOCK_SUCCESS = "OK";
    private static final String SET_IF_NOT_EXIST = "NX";
    private static final String SET_WITH_EXPIRE_TIME = "PX";

    /**
     * 尝试获取分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @param expireTime 超期时间
     * @return 是否获取成功
     */
    public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String requestId, int expireTime) {

        String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);

        if (LOCK_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }
}
```

可以看到，加锁就一行代码：`jedis.set(String key, String value, String nxxx, String expx, int time)`，这个set()方法一共有5个参数：

- key，使用`key`作为锁标识，因为key是唯一的。
- value，传入请求的唯一标识，比如`requestId`，线程`id`等，用于区分不同的请求。但有`key`作为锁就够了，为什么还要用到`value`？因为在删除锁时，可以通过`value`作为判断加锁和删锁是否为同一请求，避免错误删除。
- nx，参数传入`NX`，意思是SET IF NOT EXIST，即当`key`不存在时，进行set操作；若`key`已经存在，不做任何操作；
- px，参数传入`PX`，给key设置一个过期的时间，具体时间由第5个参数决定。
- time，`key`的过期时间。

以上的`set()`方法就只会导致两种结果：1. 当前没有锁（`key`不存在），那么就进行加锁操作，并给锁设置个有效时间，同时`value`保存请求的唯一标识。2. 已有锁存在，不做任何操作。

##### 错误示例1

常见的错误操作是使用`jedis.setnx()`和`jedis.expire()`组合实现加锁，代码如下：

```java
public static void getLock(Jedis jedis, String lockKey, String requestId, int expireTime) {

    Long result = jedis.setnx(lockKey, requestId);
    if (result == 1) {
        // 若在这里程序突然崩溃，则无法设置过期时间，锁将永远存在，导致死锁
        jedis.expire(lockKey, expireTime);
    }
}
```

由于`setnx`和`expire`是两条`Redis`命令，不具有原子性，如果线程在执行完`setnx()`之后突然崩溃，导致锁没有设置过期时间，那么将会发生死锁。以前这样实现，是因为低版本的`jedis`不支持多参数的`set()`方法。

##### 错误示例2

```java
public static boolean getLock(Jedis jedis, String lockKey, int expireTime) {

    long expires = System.currentTimeMillis() + expireTime;
    String expiresStr = String.valueOf(expires);

    // 如果当前锁不存在，返回加锁成功
    if (jedis.setnx(lockKey, expiresStr) == 1) {
        return true;
    }

    // 如果锁存在，获取锁的过期时间
    String currentValueStr = jedis.get(lockKey);
    if (currentValueStr != null && Long.parseLong(currentValueStr) < System.currentTimeMillis()) {
        // 锁已过期，获取上一个锁的过期时间，并设置现在锁的过期时间
        String oldValueStr = jedis.getSet(lockKey, expiresStr);
        if (oldValueStr != null && oldValueStr.equals(currentValueStr)) {
            // 考虑多线程并发的情况，只有一个线程的设置值和当前值相同，它才有权利加锁
            return true;
        }
    }
        
    // 其他情况，一律返回加锁失败
    return false;

}
```

这一种错误示例比较难以发现问题，实现也比较复杂。实现思路：使用`jedis.setnx()`命令实现加锁，其中`key`是锁，`value`是锁的过期时间。执行过程：

1. 通过`setnx()`方法尝试加锁，如果锁不存在，返回加锁成功
2. 如果锁已经存在，则获取锁的过期时间和当前时间比较，如果锁已经过期，则设置新的过期时间，返回加锁成功。

这段代码的问题：

1. 由于是客户端自己生成过期时间，所以需要强制要求分布式下每个客户端的时间必须同步。
2. 当锁过期的时候，如果多个客户端同时执行`jedis.getSet()`方法，那么虽然最终只有一个客户端可以加锁，但是这个客户端的锁的过期时间可能被其他客户端覆盖。
3. 锁不具有处理线程的标识，任何客户端都可以解锁。

### 解锁操作

##### 正确示例

```java
public class RedisTool {

    private static final Long RELEASE_SUCCESS = 1L;

    /**
     * 释放分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @return 是否释放成功
     */
    public static boolean releaseDistributedLock(Jedis jedis, String lockKey, String requestId) {

        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));

        if (RELEASE_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }
}
```

解锁只需要两行代码就搞定了，首先写了一个简单的`Lua`脚本代码，并将`Lua`代码传到`jedis.eval()`方法里，使参数`KEYS[1]`赋值为`lockKey`，`ARGV[1]`赋值为`requestId`。`eval()`方法是将`Lua`代码交给`Redis`服务端执行。

`Lua`代码的功能：首先获取锁对应的`value`值，检查是否与`requestId`相等，如果相等则删除锁（解锁）。

之所以使用`Lua`脚本，是因为：在`eval`命令执行`Lua`代码的时候，`Lua`代码将被当成一个命令去执行，并且直到`eval`命令执行完成，`Redis`才会执行其他命令。这就保证了解锁操作是原子性的，不会出现线程安全问题。

##### 错误示例1

最常见的解锁代码就是直接使用`jedis.del()`方法删除锁，这种不先判断锁的拥有者而直接解锁的方式，会导致任何客户端都可以随时进行解锁，即使这把锁不是它的。

```java
public static void getLock(Jedis jedis, String lockKey) {
    jedis.del(lockKey);
}
```

##### 错误示例2

这种解锁代码把解锁操作分成两条命令执行：

```java
public static void getLock(Jedis jedis, String lockKey, String requestId) {
    // 判断加锁与解锁是不是同一个客户端
    if (requestId.equals(jedis.get(lockKey))) {
        // 若在此时，这把锁突然不是这个客户端的，则会误解锁
        jedis.del(lockKey);
    }
}
```

问题在于如果调用`jedis.del()`方法的时候，这把锁已经不属于当前客户端了，那执行后会解除他人加的锁。比如客户端A加锁，一段时间之后客户端A解锁，在执行`jedis.del()`之前，锁过期了，此时客户端B尝试加锁成功，然后客户端A再执行`del()`方法，则将客户端B的锁给解除了。

### 问题

1.锁超时释放锁，如果超时时间设置太长，可能导致大量请求阻塞，降低系统效率；如果超时时间设置太短，获得锁的线程可能因为某些原因卡住（`GC`）还没有执行完任务，锁就自动释放了，导致其他线程获得锁成功，从而出现数据不一致的情况。这种情况可以考虑增加一个后台线程时刻检查锁是否超时，如果还没有，则增加超时时间。`redission`的分布式锁就是采用这种方式，通过增加看门狗添加超时时间。

2.针对`redis`集群环境，如果针对某个`redis master`，客户端1写入了`myLock`这种锁`key`的`value`，此时会异步复制给对应的`master slave`实例。但是这个过程中一旦发生`redis master`宕机，主备切换，`redis slave`变为了`redis master`。接着就会导致，客户端2来尝试加锁的时候，在新的`redis master`上完成了加锁，而客户端1也以为自己成功加了锁。此时就会导致多个客户端对一个分布式锁完成了加锁，从而出现问题。这个可以使用`redission`的`redlock`解决。

## Redission普通分布式锁

### Redission简介

`Redisson`是架设在`Redis`基础上的一个`Java`驻内存数据网格框架，充分利用`Redis`键值数据库提供的一系列优势，基于Java使用工具包中常用接口，为使用者提供了一系列具有分布式特性的常用工具类。

`Redisson`使得原本作为协调单机多线程并发程序的工具包获得了协调分布式多机多线程并发系统的能力，大大降低了设计和研发大规模分布式系统的难度。同时结合各富特色的分布式服务, 更进一步简化了分布式环境中程序相互之间的协作。

### Redission分布式锁原理

![0.png](https://i.loli.net/2021/04/16/qwIgoOFy9bVn6AJ.png)

##### 加锁机制

如果某个客户端要加锁，面对的是一个`redis cluster`集群，首先会根据`hash`节点选择一台机器去获取锁，获取成功后执行`lua`脚本，保存数据到`redis`。

如果获取失败则一直通过`while`循环尝试获取锁，获取成功后，执行`lua`脚本，保存数据到`redis`。

##### watch dog自动延期机制

如果负责储存这个分布式锁的`Redisson`节点宕机，但锁正好处于锁住状态时，就出现锁死的状态。为了避免这种情况发生，`Redisson`内部提供了一个监控锁的看门狗，它的作用是在`Redisson`实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗的检查锁的超时时间是30秒钟，也可以通过修改`Config.lockWatchdogTimeout`来另行指定。

另外，`Redisson`还通过加锁的方法提供了`leaseTime`的参数来指定加锁的时间。超过这个时间后锁便自动解开了。

##### 可重入锁机制

`Redisson`可以实现可重入锁，主要原理：用`key`保存线程信息（比如线程id），`value`保存加锁次数。当执行加锁操作时，通过`key`判定是不是同一个线程，如果是加锁成功，并且`value`加1，释放锁时，`value`减1。

### Redission分布式锁使用

##### Redis几种架构

`Redis`几种常见的部署架构包括：

- 单机模式

- 主从模式

- 哨兵模式

- 集群模式

##### 单机模式

代码如下：

```java
// 构造redisson实现分布式锁的Config
Config config = new Config();
config.useSingleServer().setAddress("redis://192.168.0.6:6379").setPassword("123456").setDatabase(0);
// 构造RedissonClient
RedissonClient redissonClient = Redisson.create(config);
// 设置锁名称
RLock disLock = redissonClient.getLock("mylock");
boolean isLock;
try {
    // 尝试获取分布式锁
    isLock = disLock.tryLock(500, 15000, TimeUnit.MILLISECONDS);
    if (isLock) {
        //TODO if get lock success, do something;
        Thread.sleep(15000);
    }
} catch (Exception e) {
} finally {
    // 解锁
    disLock.unlock();
}
```

##### 哨兵模式

即`sentinel`模式，代码如下：

```java
Config config = new Config();
config.useSentinelServers().addSentinelAddress(
        "redis://192.168.0.5:6379", "redis://192.168.0.6:6379", "redis://192.168.0.7:6379")
        .setMasterName("mymaster")
        .setPassword("123456").setDatabase(0);
```

##### 集群模式

代码如下：

```java
Config config = new Config();
config.useClusterServers().addNodeAddress(
        "redis://192.168.0.5:6379", "redis://192.168.0.6:6379", "redis://192.168.0.7:6379",
        "redis://192.168.0.8:6379", "redis://192.168.0.9:6379", "redis://192.168.0.10:6379")
        .setPassword("123456").setScanInterval(5000);
```

无论那种架构，普通分布式锁的实现原理就是`Redis`通过`EVAL`命令执行`LUA`脚本。

## Redlock分布式锁

以上`Redission`分布式锁的问题：

假如客户端1对某个`master`节点写入了`redisson`锁，此时会异步复制给对应的`slave`节点。但是这个过程中，`master`节点一旦宕机，主备切换，`slave`节点从变为了 `master`节点。这时客户端2来尝试加锁的时候，在新的`master`节点上也能加锁成功，此时就会导致多个客户端对同一个分布式锁完成了加锁。所以，这个是`redis cluster`，或者是`redis master-slave`架构的主从异步复制导致的`Redis`分布式锁的最大缺陷。

由于存在以上问题，`Redis`作者`antirez`基于分布式环境下提出了一种更高级的分布式锁的实现方式：`Redlock`。

#### Redlock原理

`antirez`提出的`redlock`算法大概是这样的：

在`Redis`的分布式环境中，假设有N个`Redis master`。这些节点完全互相独立，不存在主从复制或者其他集群协调机制。然后，在N个实例上使用与在单实例下使用相同的方法获取和释放锁。假如有5个`Redis master`组成的`cluster`，为了取到锁，客户端操作的大致流程如下：

- 获取当前时间。

- 按顺序依次向5个`Redis`节点执行获取锁的操作。这个获取操作跟前面基于单`Redis`节点的获取锁的过程相同。当向`Redis`请求获取锁时，客户端应设置一个网络超时时间，这个超时时间（毫秒级）应该远小于锁的失效时间（秒级）。如果服务器端没有在规定时间内响应，客户端应尽快尝试去另外一个`Redis`实例请求获取锁。
- 客户端使用当前时间减去开始获取锁的时间（步骤1记录的时间）得到锁使用的时间。当且仅当从大多数（N/2+1，这里是3个节点）的`Redis`节点都获取到锁，并且使用的时间小于锁的失效时间时，才算加锁成功。
- 客户端获取锁成功后，锁的有效时间等于最初设置的有效时间减去获取锁消耗的时间。
- 如果客户端获取锁失败（没有在至少N/2+1个`Redis`实例获取到锁，或者获取锁的时间超过了有效时间），那么客户端应立即向所有`Redis`节点发起释放锁的操作（即前面介绍的单机`Redis Lua`脚本释放锁的方法），防止某些节点获取到锁导致其他客户端不能获取锁。

#### Redlock加锁代码

`redlock`的核心变化是使用了`RedissonRedLock`类。这个类是`RedissonMultiLock`的子类，调用了`tryLock`方法，即调用了`RedissonMultiLock`的`tryLock`方法，代码如下：

```java
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    // 实现要点之允许加锁失败节点限制
    int failedLocksLimit = failedLocksLimit();
    List<RLock> acquiredLocks = new ArrayList<RLock>(locks.size());
    // 实现要点之遍历所有节点通过EVAL命令执行lua加锁
    for (ListIterator<RLock> iterator = locks.listIterator(); iterator.hasNext();) {
        RLock lock = iterator.next();
        boolean lockAcquired;
        try {
            // 对节点尝试加锁
            lockAcquired = lock.tryLock(awaitTime, newLeaseTime, TimeUnit.MILLISECONDS);
        } catch (RedisConnectionClosedException|RedisResponseTimeoutException e) {
            // 如果抛出这类异常，为了防止加锁成功，但是响应失败，需要解锁
            unlockInner(Arrays.asList(lock));
            lockAcquired = false;
        } catch (Exception e) {
            // 抛出异常表示获取锁失败
            lockAcquired = false;
        }
        
        if (lockAcquired) {
            // 成功获取锁集合
            acquiredLocks.add(lock);
        } else {
            // 如果达到了允许加锁失败节点限制，那么break，即此次Redlock加锁失败
            if (locks.size() - acquiredLocks.size() == failedLocksLimit()) {
                break;
            }               
        }
    }
    return true;
}
```

这段源码就是`Redlock`算法的实现。以`sentinel`模式架构为例，有`sentinel-1`，`sentinel-2`，`sentinel-3`总计3个`sentinel`模式集群，如果要获取分布式锁，那么需要向这3个`sentinel`集群通过`EVAL`命令执行`LUA`脚本，需要3/2+1=2，即至少2个`sentinel`集群响应成功，才算成功的以`Redlock`算法获取到分布式锁。

#### Redlock使用

代码如下：

```java
Config config = new Config();
config.useSentinelServers().addSentinelAddress("192.168.0.5:6379","192.168.0.6:6379", "192.168.0.7:6379")
        .setMasterName("masterName")
        .setPassword("password").setDatabase(0);
RedissonClient redissonClient = Redisson.create(config);
// 还可以getFairLock(), getReadWriteLock()
RLock redLock = redissonClient.getLock("REDLOCK_KEY");
boolean isLock;
try {
    isLock = redLock.tryLock();
    // 500ms拿不到锁, 就认为获取锁失败。10000ms即10s是锁失效时间。
    isLock = redLock.tryLock(500, 10000, TimeUnit.MILLISECONDS);
    if (isLock) {
        //TODO if get lock success, do something;
    }
} catch (Exception e) {
} finally {
    // 解锁
    redLock.unlock();
}
```

#### RedLock问题讨论

- 有`redis`节点发生崩溃后重启

假设一共有5个`Redis`节点：A, B, C, D, E。出现如下事件：

1、客户端1成功锁住了A, B, C，获取锁成功（但D和E没有锁住）。

2、节点C崩溃重启了，但客户端1在C上加的锁没有持久化下来，丢失了。

3、节点C重启后，客户端2锁住了C, D, E，获取锁成功。

4、客户端1和客户端2同时获得了锁。

为了应对这一问题，`antirez`又提出了延迟重启(delayed restarts)的概念。也就是说，一个节点崩溃后，先不立即重启它，而是等待一段时间再重启，这段时间应该大于锁的有效时间(lock validity time)。这样的话，这个节点在重启前所参与的锁都会过期，它在重启后就不会对现有的锁造成影响。

- 客户端长期阻塞导致锁过期

![1.png](https://i.loli.net/2021/04/16/6qfCwILvhRSAjQ1.png)

如上图，客户端1在获得锁之后发生了很长时间的`GC pause`，在此期间，它获得的锁过期了，而客户端2获得了锁。当客户端1从`GC pause`中恢复过来时，它不知道自己持有的锁已经过期了，它依然向共享资源（上图中是一个存储服务）发起了写数据请求，而这时锁实际上是被客户端2持有的，因此两个客户端的写请求就有可能冲突。

为了解决这个问题，引入了`fencing token`的概念：

![2.png](https://i.loli.net/2021/04/16/mHjbMJ1eq9hUrWP.png)

如上图，客户端1先获取到的锁，生成一个较小的`fencing token`，值为33，而客户端2后获取到锁，有一个较大的`fencing token`，值为34。客户端1从`GC pause`中恢复过来之后，依然是向存储服务发送访问请求，但是带了`fencing token = 33`。存储服务发现它之前已经处理过34的请求，所以会拒绝掉这次值为33的请求。这样就避免了冲突。

但这已经超出了`Redis`实现分布式锁的范围，单纯用`Redis`没有命令来实现生成`Token`。

- 时钟跳跃问题

假设有5个`Redis`节点A, B, C, D, E。出现如下事件序列：

1、客户端1从`Redis`节点A, B, C成功获取了锁（多数节点）。由于网络问题，与D和E通信失败。

2、节点C上的时钟发生了向前跳跃，导致它上面维护的锁快速过期。

3、客户端2从`Redis`节点C, D, E成功获取了同一个资源的锁（多数节点）。

4、客户端1和客户端2现在都认为自己持有了锁。

这个问题用`Redis`实现分布式锁暂时无解，而生产环境这种情况是存在的。

## 总结

- 失效时间如何设置

这个问题的场景是，假设设置失效时间10秒，如果由于某些原因导致10秒还没执行完任务，这时候锁自动失效，导致其他线程也会拿到分布式锁。

这确实是Redis分布式最大的问题，不管是普通分布式锁，还是`Redlock`算法分布式锁，都没有解决这个问题。也有一些文章提出了对失效时间续租，即延长失效时间，这又提升了分布式锁的复杂度。

- zookeeper/redis选择

没有绝对的好与坏，只有更适合业务的选择。就性能而言，`redis`明显优于`zookeeper`；就分布式锁实现的健壮性而言，`zookeeper`很明显优于`redis`。如何选择，取决于业务。

`Redis`并不能实现严格意义上的分布式锁。但是这并不意味着上面的方案一无是处。需要结合具体的应用场景考虑，如果对数据一致性要求不高，只是为了提高效率，协调各个客户端避免做重复的工作，即使锁失效了，也不会产生严重后果。但是如果应用场景需要很强的数据正确性，那么用`Redis`实现分布式锁并不合适，可能各种各样的问题，且解决起来就很复杂，为了正确性，需要使用`zab`、`raft`共识算法，或者使用带有事务的数据库来实现严格意义上的分布式锁。