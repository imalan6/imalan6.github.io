# Redis+Token实现接口幂等性

## 幂等性

### 概念

任意多次请求执行所产生的效果与一次执行的效果相同。也就是说对系统的影响只能是一次性的，不能重复执行处理。但在实际项目环境中，一个对外服务的接口可能会面临很多次重复的请求，比如：

- 前端重复提交。用户多次点击提交按钮，导致重复提交数据。
- 消息重复消费。一般指的是消息中间件，如`RabbitMQ`，由于网络抖动，`MQ Broker`将消息发送给消费端消费，消费端进行了消费，但在返回`ACK`给`MQ Broker`时网络中断，导致`MQ Broker`认为消费端没能正常消费消息，`MQ Broker`重发消息。
- 页面回退再次提交。比如，用户购买商品时，如果第一次下单成功，这时候用户点击浏览器返回按钮，返回上一个下单页面，再次点击下单，就会导致重复提交。

- 网络延迟或阻塞。微服务之间通信出现网络延迟或阻塞，导致请求失败，如`feign`可能触发重试机制，这就可能导致被调用服务收到多份请求数据；

为了避免以上这些情况，都要求后端接口必须实现幂等性，能够对重复的请求数据进行处理，不然可能出现业务问题。

### 解决方案

实现接口幂等性通常有如下方案：

- 数据库唯一索引。在数据表中，为请求数据的唯一标识建立唯一索引，可以防止表中插入相同数据。比如：在订单表中为订单id建立唯一索引。这样，即使有重复的订单提交请求，也只能插入一条。
- 防重数据表。使用请求数据的唯一字段作为去重表的唯一索引，把唯一字段插入去重表后再进行业务操作，并保证操作都在同一事务中。因为去重表有唯一约束，如果是重复请求，会插入失败，解决了幂等问题。这里要注意的是，去重表和业务表应该在同一个数据库中，这样保证在同一个事务里，即使业务操作失败，也会把去重表的数据进行回滚。保证了数据的一致性。

- `Token`机制。`Token`机制是目前应用较多的接口幂等性方案。每次接口请求前先获取一个`Token`，然后在请求业务时带上这个`Token`，服务端进行验证，如果验证通过删除`Token`，下次请求再次判断`Token`。详见第二部分。

- `Redis`的`SETNX`防重。服务端将请求的唯一标识用`SETNX`方式存入`Redis`，并设置超时时间。如果存入成功，则继续做后续业务请求；如果存入失败，则代表已经执行过当前请求。

- 数据库乐观锁。数据库中增加版本号字段，每次更新通过版本号来判断。如果有线程已经更新了version字段，其余线程执行sql操作就会失败。方式如下：

  ```sql
  #客户端请求服务端，首先查询当前的version版本
  select version from … where …
  
  #根据version版本执行操作
  UPDATE … SET … version=(version+1) WHERE … AND version=version
  ```

- 数据库悲观锁。执行select加上for update语句，其他并发线程执行时会发生互斥，需要等待行锁释放，从而防止同一条数据被重复线程处理。

  ```sql
  START TRANSACTION; #开启事务
  SELETE * FROM TABLE WHERE ... FOR UPDATE;
  UPDATE TABLE SET ... WHERE ...;
  COMMIT; #提交事务
  ```

- 业务层锁机制。如果多个线程可能在同一时间处理相同的数据，比如多个线程在同一时刻都拿到了相同的数据处理，可以加锁访问（根据是否分布式系统采用`JVM`锁或者分布式锁），处理完成后释放锁。获取到锁的先查询数据库判断数据是否已经被处理过。

## Redis+Token机制

### 实现原理

`Token`机制配合`Redis`处理接口幂等性问题是业界用的比较多的一种方式。简单理解，就是每次请求都拿着一张门票，这个门票是一次性的，用过一次就被毁掉了，不能重复利用。`Token`令牌就相当于门票的概念，每次请求业务接口之前先获取`Token`，然后访问业务接口时带上刚获取的`Token`令牌。服务器处理请求的时候校验`Token`，校验通过就继续处理业务，否则返回`error`。这个`Token`只能用一次，如果客户端使用相同的令牌再次请求，那么就不处理，直接返回。大致流程如下图所示：

![0.png](https://i.loli.net/2021/03/31/uMws5x8qyHvQIZc.png)

**主要步骤如下：**

1）用户端根据业务类型，调用`Token`接口获取相应`Token`；

2）服务端生成`Token`，并把`Token`保存到`redis`中，然后向用户端返回`Token`；

3）用户端携带`Token`访问业务接口，`Token`一般在请求参数或者请求头中传递；

4）服务端判断`Token`是否存在`Redis`中，如果存在表示是第一次请求，然后删除`Token`，继续执行下面的业务处理；

5）用户端如果短时间内重复提交请求，因为两次请求携带的`Token`是一样的，所以第二次请求时，服务器校验`Token`时，`Redis`中的`Token`已经被删除掉，这就表示是重复操作，所以第二次请求会校验失败，这就保证了重复请求不会被执行；

**注意：**

不过`Token`方案也需要注意：

- 是先删除`Token`再处理业务，还是先处理业务再删除`Token`。

如果先删除`Token`再处理业务，可能导致删除了`Token`，但是业务还没有执行，服务器崩溃了。由于防重设计导致，其他请求都不能执行；

如果先处理业务再删除`Token`，可能导致业务处理成功，但删除`Token`失败，防重设计失效。其他线程可能继续执行业务，导致业务被重复执行，后果很严重；

考虑到对业务影响的严重级别，最好设计为先删除`Token`再处理业务。如果业务处理失败，用户端重新获取`Token`再次请求处理。

- 获取`Token`，比较和删除操作必须保证原子性。

`redis.get(token)`，`token.equals()`，`redis.del(token)`，获取/比较/删除，这三个操作并不是原子性的，可能导致高并发情况下，多个线程同时获取并处理到同样的数据。可以在redis中使用lua脚本保证操作的原子性。

```lua
if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end
```

也可以采用锁机制，以`Token`字段上锁，比如`synchronized(Token)`，或者分布式锁。既互斥访问，同时也兼顾性能。

### 代码实现

实现一个方法添加`@Idempotent`注解后自动进行幂等性处理。

1）Token服务接口

创建一个`Token`服务接口，包含两个方法：创建`token`和验证`token`。创建`token`采用随机生成的字符串，而验证`token`就是从请求中取出`token`，检查并删除`redis`中对应的`token`。代码如下：

```java
public interface TokenService {

    /**
     * 创建token
     * @return
     */
    public  String createToken();

    /**
     * 检验token
     * @param request
     * @return
     */
    public boolean checkToken(HttpServletRequest request) throws Exception;

}
```

服务实现类：

```java
@Service
public class TokenServiceImpl implements TokenService {

    @Autowired
    private RedisService redisService;

    /**
     * 创建token
     *
     * @return
     */
    @Override
    public String createToken() {
        String str = RandomUtil.randomUUID();
        StrBuilder token = new StrBuilder();
        try {
            token.append(Constant.Redis.TOKEN_PREFIX).append(str);
            //设置过期时间为600秒,具体视业务而定
            redisService.setEx(token.toString(), token.toString(), 600L);
            boolean notEmpty = StrUtil.isNotEmpty(token.toString());
            if (notEmpty) {
                return token.toString();
            }
        }catch (Exception ex){
            ex.printStackTrace();
        }
        return null;
    }


    /**
     * 检验token
     *
     * @param request
     * @return
     */
    @Override
    public boolean checkToken(HttpServletRequest request) throws Exception {

        String token = request.getHeader(Constant.TOKEN_NAME);
        if (StrUtil.isBlank(token)) {// header中不存在token
            token = request.getParameter(Constant.TOKEN_NAME);
            if (StrUtil.isBlank(token)) {// parameter中也不存在token
                throw new ServiceException(Constant.ResponseCode.ILLEGAL_ARGUMENT, 100);
            }
        }

        if (!redisService.exists(token)) {
            throw new ServiceException(Constant.ResponseCode.REPETITIVE_OPERATION, 200);
        }

        boolean remove = redisService.remove(token);
        if (!remove) {
            throw new ServiceException(Constant.ResponseCode.REPETITIVE_OPERATION, 200);
        }
        return true;
    }
}
```

3）自定义注解

自定义一个注解，把它添加在需要实现幂等的方法上，凡是添加了该注解的方法，都会实现幂等性检查。

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Idempotent {
  
}
```

4）配置`SpringMVC`拦截器，对添加了注解方法的请求进行拦截处理。

```java
/**
 * 拦截器
 */
@Component
public class IdempotentInterceptor implements HandlerInterceptor {

    @Autowired
    private TokenService tokenService;

    /**
     * 预处理
     *
     * @param request
     * @param response
     * @param handler
     * @return
     * @throws Exception
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        if (!(handler instanceof HandlerMethod)) {
            return true;
        }
        HandlerMethod handlerMethod = (HandlerMethod) handler;
        Method method = handlerMethod.getMethod();
        //检查是否存在Idempotment注解
        AutoIdempotent methodAnnotation = method.getAnnotation(Idempotent.class);
        if (methodAnnotation != null) {
            try {
                // 幂等性校验, 校验通过则放行, 校验失败则抛出异常, 并通过统一异常处理返回友好提示
                return tokenService.checkToken(request);
            }catch (Exception ex){
                ResultVo failedResult = ResultVo.getFailedResult(101, ex.getMessage());
                writeReturnJson(response, JSONUtil.toJsonStr(failedResult));
                throw ex;
            }
        }
        //必须返回true,否则会被拦截一切请求
        return true;
    }


    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

    }

    /**
     * 返回的json值
     * @param response
     * @param json
     * @throws Exception
     */
    private void writeReturnJson(HttpServletResponse response, String json) throws Exception{
        PrintWriter writer = null;
        response.setCharacterEncoding("UTF-8");
        response.setContentType("text/html; charset=utf-8");
        try {
            writer = response.getWriter();
            writer.print(json);

        } catch (IOException e) {
        } finally {
            if (writer != null)
                writer.close();
        }
    }

}
```

5）创建配置类将`IdempotentInterceptor`拦截器注册到拦截器链中

```java
@Configuration
public class WebConfiguration extends WebMvcConfigurerAdapter {

    @Resource
   private IdempotentInterceptor idempotentInterceptor;

    /**
     * 添加拦截器
     * @param registry
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(idempotentInterceptor);
        super.addInterceptors(registry);
    }
}
```

6）创建一个controller类

```java
@RestController("/test")
public class TestController {

    @Resource
    private TokenService tokenService;

    @PostMapping("/token")
    public String  getToken(){
        String token = tokenService.createToken();
        if (StrUtil.isNotEmpty(token)) {
            ResultVo resultVo = new ResultVo();
            resultVo.setCode(Constant.code_success);
            resultVo.setMessage(Constant.SUCCESS);
            resultVo.setData(token);
            return JSONUtil.toJsonStr(resultVo);
        }
        return StrUtil.EMPTY;
    }


    @Idempotent
    @PostMapping("/idempotence")
    public String testIdempotence() {
		return "success";
    }
}
```

7）测试

首先，使用`postman`发送请求，访问`/test/token`接口获取到`token`。

然后，再使用`postman`访问`/test/idempotence`接口，并把上一步获取到的`token`放到请求的`header`中。可以看到返回`success`字符串，表示访问成功。

最后，再重复上一步，发送多次请求，提示访问失败，表示重复请求已被防重设计拦截。











































