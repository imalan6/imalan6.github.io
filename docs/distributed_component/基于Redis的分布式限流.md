# 基于Redis的分布式限流

## 简介

在开发服务接口时，为了保护服务器的资源，防止客户端对接口的滥用， 通常来说会限制服务接口的调用次数。比如限制用户在一段时间内，比如1分钟，调用服务接口的次数不能大于某个值，比如说100次。如果用户调用接口的次数超过上限的话，就直接拒绝用户请求，返回错误信息。

服务接口的流量控制策略包括：分流、降级、限流等。限流策略，虽然降低了服务接口的访问频率和并发量，却换回了服务接口和业务应用系统的高可用。

## 常用限流算法

#### 漏桶算法

漏桶(Leaky Bucket)算法思路很简单，水(请求)先进入到漏桶里，漏桶以一定的速度出水(接口有响应速率)。当水流入速度过大会直接溢出(访问频率超过接口响应速率)，然后拒绝请求，可以看出漏桶算法能强行限制数据的传输速率。如下图所示：

![2.png](https://i.loli.net/2021/04/14/2xQsgikoZblnX3Y.png)

 包括两个变量：

- 桶的大小，支持流量突发增多时可以存多少的水(burst)

- 水桶漏洞的大小(rate)

因为漏桶的漏出速率是固定的参数，所以，即使网络中不存在资源冲突(没有发生拥塞)，漏桶算法也不能使流突发(burst)到端口速率。因此，漏桶算法不适合存在突发流量特性的应用场景。

#### 令牌桶算法

令牌桶算法(Token Bucket)是和漏桶算法效果一样，但方向相反的算法。令牌桶算法：随着时间流逝，系统会按恒定1/QPS时间间隔(比如QPS=100，则间隔为10ms)往桶里加入`Token`(想象和漏洞漏水相反，有个水龙头不断地往里面加水)。如果桶满了就不再添加了。新请求来临时，会各自拿走一个`Token`，如果没有`Token`可拿就阻塞或拒绝服务。

![3.jpg](https://i.loli.net/2021/04/14/P8qWoRNHM5OXlet.jpg)

令牌桶的好处是可以方便的改变速度，一旦需要提高速率，则按需提高放入桶中的令牌的速率。一般会定时(比如100毫秒)往桶中增加一定数量的令牌，有些变种算法则实时计算应该增加的令牌的数量。

## 分布式限流组件

一般限流方案只能针对于单个`JVM`有效，也就是单机方案。而对于分布式应用需要一个分布式限流的方案。针对分布式限流，写了这个组件：

[https://gitee.com/shuimenjian/ala-distributed-ratelimit](https://gitee.com/shuimenjian/ala-distributed-ratelimit)

主要原理

假设一个用户（用IP判断）每秒访问某服务接口的次数不能超过10次，那么可以在`Redis`中创建一个键，并设置键的过期时间为60秒。当一个用户对此服务接口发起一次访问就把值加1，在单位时间（此处为1s）内当键值增加到10的时候，就禁止访问服务接口。

## 核心代码

#### Lua脚本

**脚本作用**

- 减少网络开销: 不使用`Lua`的代码需要向`Redis`发送多次请求, 而脚本只需一次即可, 减少网络传输

- 原子操作: `Redis`将整个脚本作为一个原子执行, 无需担心并发, 也就无需事务

- 复用: 脚本会永久保存`Redis`中, 其他客户端可继续使用

- `Redis`添加了对`Lua`的支持，能够很好的满足原子性、事务性的支持，免去了很多的异常逻辑处理

**脚本内容**

是整个限流操作的核心，通过执行`Lua`脚本进行限流的操作。脚本内容如下：

```lua
    --获取KEY
    local key1 = KEYS[1]

    local val = redis.call('incr', key1)
    local ttl = redis.call('ttl', key1)

    --获取ARGV内的参数并打印
    local expire = ARGV[1]
    local times = ARGV[2]

    redis.log(redis.LOG_DEBUG,tostring(times))
    redis.log(redis.LOG_DEBUG,tostring(expire))

    redis.log(redis.LOG_NOTICE, "incr "..key1.." "..val);
    if val == 1 then
        redis.call('expire', key1, tonumber(expire))
    else
        if ttl == -1 then
            redis.call('expire', key1, tonumber(expire))
        end
    end

    if val > tonumber(times) then
        return 0
    end

    return 1
```

- 首先脚本获取Java代码中传递而来的要限流的模块的`key`，不同的模块`key`值一定不能相同，否则会覆盖

- `redis.call('incr', key1)`对传入的key做`incr`操作，如果`key`首次生成，设置超时时间`ARGV[1]`（初始值为1）

- `ttl`是为防止某些`key`在未设置超时时间并长时间已经存在的情况下做的保护的判断

- 每次请求都会做+1操作，当限流的值大于注解的阈值，则返回0，表示已经超过请求限制，触发限流，否则为正常请求

**脚本调用**

调用并执行`Lua`脚本代码：

```java
getRedisScript = new DefaultRedisScript<>();
getRedisScript.setResultType(Long.class);
getRedisScript.setScriptSource(new ResourceScriptSource(new ClassPathResource("alaRateLimit.lua")));
        
// 调用Lua脚本
Long result = (Long) redisTemplate.execute(getRedisScript, keyList, expireTime, limitTimes);
if (result == 0) {
	//表示需要限流
}
```

#### 自定义注解

定义一个方法注解`@AlaRateLimit`：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AlaRateLimit {

    /**
     * 限流key，区分不同的方法
     * @return
     */
    String key() default "";

    /**
     * 限流次数
     * @return
     */
    long limit() default 100;

    /**
     * 限流间隔时间
     * @return
     */
    long interval() default 60;

    /**
     * 限流返回，消息形式
     * @return
     */
    String message() default "rate limit error";


    /**
     * 限流返回，对象形式，优先级高于message
     * @return
     */
    Class<?> fallback() default void.class;
}
```

使用`AOP`处理注解，核心代码：

```java
@Aspect
@Component
public class RateLimitHandler {

    @Pointcut("@annotation(com.alan6.distributed.ratelimit.annotation.AlaRateLimit)")
    public void rateLimit() {

    }

    @Around("@annotation(rateLimit)")
    public Object around(ProceedingJoinPoint proceedingJoinPoint, AlaRateLimit rateLimit) throws Throwable {

        Signature signature = proceedingJoinPoint.getSignature();
        if (!(signature instanceof MethodSignature)) {
            throw new IllegalArgumentException("the Annotation @AlaRateLimit must be used on method");
        }

        // 限流模块key
        String limitKey = rateLimit.key();

        if (StringUtils.isBlank(limitKey)) {
            // 如果没有设置key，key的默认值=方法包名+方法名
            limitKey = proceedingJoinPoint.getSignature().getDeclaringType().getPackage().getName() + "." + proceedingJoinPoint.getSignature().getName();
        }

        // 限流阈值
        long limitTimes = rateLimit.limit();
        // 限流超时时间
        long expireTime = rateLimit.interval();
        // 限流提示语
        String message = rateLimit.message();
        // 获取降级对象
        Object fallback = rateLimit.fallback().newInstance();
        List<String> keyList = new ArrayList();
        // 设置key值为注解中的值
        keyList.add(limitKey);
        
        // 调用Lua脚本
        Long result = (Long) redisTemplate.execute(getRedisScript, keyList, expireTime, limitTimes);
        if (result == 0) {
            // 指定了降级
            if (fallback != null) {
                // 检查降级方法是否实现AlaRateLimitFallback接口
                Assert.isTrue(
                        !rateLimit.fallback().getSuperclass().isInterface(),
                        "Rate limit fallback class must implement the interface AlaRateLimitFallback"
                );

                // 反射调用降级方法
                Method m = fallback.getClass().getMethod("fallbackResult", null);

                // 反射执行方法
                return m.invoke(fallback, null);

            }else {
                return message;
            }
        }
        return proceedingJoinPoint.proceed();
    }
}

```

主要逻辑就是：解析注解，获取注解参数，调用`Lua`脚本对`key`进行`incr`操作，如果`interval`时间内，数量达到限制阈值，则调用降级方法处理或返回`error`消息。

#### 降级方法

定义一个降级接口：

```java
public interface AlaRateLimitFallback {
    public Object fallbackResult();
}
```

降级方法需要实现降级接口，实现`fallbackResult()`方法，给出一个`demo`：

```java
public class RatelimitFallback implements AlaRateLimitFallback {
    @Override
    public Object fallbackResult() {
        return new BaseResult(888, "rate limit error");
    }
}
```

## 使用方法

#### 注解说明

在需要限流的方法上添加`@AlaRateLimit`注解，注解参数含义如下：

- `key`：限流`key`，用于区分需要限流的不同方法。没有指定的话，默认为注解所在包名 + '.' + 方法名

- `limit`：限流次数，默认100次

- `interval`：限流间隔时间，默认60s

- `message`：限流后返回字符串形式的`error`消息，默认"`rate limit error`"

- `fallback`： 限流后返回对象形式，优先级高于`message`，需要实现`AlaRateLimitFallback`接口

#### 使用示例

1、实现降级处理接口

```java
public class RatelimitFallback implements AlaRateLimitFallback {
    @Override
    public Object fallbackResult() {
        return new BaseResult(888, "rate limit error");
    }
}
```

2、在方法上添加注解

```java
@RestController
@RequestMapping("/test")
public class TestController {

    @AlaRateLimit(limit = 10, interval = 10, fallback = RatelimitFallback.class)
    @GetMapping("/limit")
    public Object testRateLimit(){
        return "ok";
    }
}
```

## 测试

在10s秒内，连续请求`/test/limit`接口，当次数小于等于10次时，接口正常返回`String`消息“`ok`”

![1.png](https://i.loli.net/2021/04/14/qLZEBJ7iROUP246.png)

在10s秒内，连续请求/test/limit接口，当次数大于10次时，调用次数超过限制，触发降级处理，接口返回`json`消息"`{"code":888,"message":"rate limit error"}`"

![0.png](https://i.loli.net/2021/04/14/iuk74ZqpVfHL3Gg.png)

