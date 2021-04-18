# SpringCloud系列之Zuul

## 服务网关

服务网关是统一管理`API`的一个网络关口、通道，是整个微服务平台所有请求的唯一入口，所有的客户端和消费端都通过统一的网关接入微服务，在网关层处理所有的非业务功能。

- **路由转发**：接收所有外界请求，转发给微服务处理；

- **过滤处理**：对请求进行处理，比如权限校验、限流以及监控等。

`Zuul`是`Netflix OSS`中的一员，是一个基于`JVM`路由和服务端的负载均衡器。提供路由、监控、弹性、安全等方面的服务框架。`Zuul`能够与`Eureka`、`Ribbon`、`Hystrix`等组件配合使用。`Zuul`的核心是过滤器，通过这些过滤器可以扩展出很多功能，比如：

- **动态路由**

动态地将客户端的请求转发到后端具体的服务，完成业务逻辑处理。

- **认证鉴权**

对所有用户请求做身份认证，拒绝非法请求，提高整个系统安全性。

- **预警监控**

可以对所有用户请求进行监控，记录详细的请求响应日志，实时统计当前系统访问量、状态。

- **负载均衡**

为每种类型的请求分配容量并丢弃超过限额的请求。

- **静态资源处理**

直接在`Zuul`处理静态资源并响应，而并非转发这些请求到内部集群中。

## 部署

- **配置**

1）`pom`文件添加依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

2）启动类加注解

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableZuulProxy
public class ZuulApplication {
    public static  void  main(String[] args){
        SpringApplication.run(ZuulApplication.class, args);
    }
}
```

3）配置文件

```yaml
server:
  port: 9000
  
spring:
  application:
    name: land-zuul

zuul:
  routes:
    land-user:
      path: /user/**
      service-id: land-user
      stripPrefix: false  #前缀方式映射-去掉前缀,不然访问需要/user前缀，比如访问/user/hello需要写成/user/user/hello.

management:
  endpoints:
    web:
      exposure:
        include: "*"  #实现智能端点，查看路由信息，filters，路径：/actuator/routes，需要使用actuator，zuul已默认集成starter-actuator

eureka:
  instance:
    prefer-ip-address: true   #开启显示IP地址
    instance-id: ${spring.cloud.client.ip-address}:${server.port}   #eureka页面显示IP地址：端口号
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

------

- **测试**

启动`zuul`和`user`服务，访问：http://127.0.0.1:9000/user/hello ，正常访问。

**需要注意：**如果使用`springboot`部署`zuul`，并且添加了`starter-security`依赖，需要注销掉，不然会跳转到默认的登录页面。

```xml
<!--		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-security</artifactId>
		</dependency>-->
```

![0.png](https://i.loli.net/2021/03/20/vmu1gxE4nliNWrk.png)

这是因为在`SpringBoot`中，默认的`Spring Security`生效了，此时所有接口的访问都是被保护的。需要通过验证才能正常访问。`Spring Security`提供了一个默认的用户，用户名是user，而密码则是启动项目的时候自动生成的。如果不想注销依赖，可以有如下方法解决：

**`SpringBoot1.x`以前，修改配置文件如下：**

```yaml
security:
  basic:
	enabled: false
```

**`SpringBoot2.x`及以后，添加配置类如下：**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
 
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //配置不需要登陆验证
        http.authorizeRequests().anyRequest().permitAll().and().logout().permitAll();
    }
}
```

## 常用基本配置

- **配置path**

上文中,我们配置的一个路由规则是这样的：

```yml
zuul:
  routes:
    land-user:
      path: /user/**
      service-id: land-user
```

这里指定了`path`和`service-id`，可以简化一下，方式如下：

```yml
zuul:
  routes:
    service-id: /user/**
```

`zuul.routes`后面是服务名，值是路径，上面这两种配置的方式都是可以的，下面这样更简洁。

- **配置url**

同时指定`path`和`url`，例如：

```yml
zuul:
  routes:
    user-route:     # 该配置方式中，user-route 只是给路由一个名称，可以任意起名。
      url: http://localhost:8001/     # 指定的 url
      path: /user/**     # url 对应的路径
```

这样就可以将 `/user/**` 映射到 `http://localhost:8001/**`。

需要注意的是，使用这种方式配置的路由不会作为`HystrixCommand`执行，也不能使用`Ribbon`来负载均衡多个`url`。

同时指定`path`和`url`，并且不破坏`Zuul`的`Hystrix`、`Ribbon`特性。配置如下：

```yml
zuul: 
  routes: 
    land-user: 
      url: http://localhost:8001/
      path: /user/**
      service-id: land-user
ribbon: 
  eureka: 
    enabled: false     # 为 Ribbon 禁用 Eureka
land-user: 
  ribbon: 
    listOfServices: localhost:8001,localhost:8002
```

- **路由前缀**

为所有路由规则增加前缀，配置如下：

```yml
zuul:
  prefix: /api
```

比如之前访问路径是 http://localhost:9000/user/hello ，配置了前缀之后，访问路径就变成了 http://localhost:9000/api/user/hello 。

- **忽略指定服务**

忽略服务可以使用`zuul.ignored-services`配置需要忽略的服务，多个用逗号分隔。例如：

```yml
zuul: 
  ignored-services: land-test, land-hi
```

这样`Zuul`就忽略掉了`land-test`和`land-hi`微服务，只代理其他微服务。

如果要忽略所有微服务，只路由指定微服务，可以将`zuul.ignored-services`设为`*`。

```yml
zuul: 
  ignored-services: '*'	 # 使用 '*' 可忽略所有微服务
  routes: 
    land-user: /user/**
```

- **通配符**

不论是使用传统配置方式还是服务路由的配置方式，都需要使用通配符方式为每个路由定义匹配表达式。

| 通配符 |               说明               |
| :----- | :------------------------------: |
| ?      |         匹配任意单个字符         |
| *      |        匹配任意数量的字符        |
| **     | 匹配任意数量的自负，支持多级目录 |

| url路径  |                             说明                             |
| :------- | :----------------------------------------------------------: |
| /user/?  |   可以匹配/user/之后的一个字符的路径，比如/user/a，/user/b   |
| /user/*  |  可以匹配/user/之后任意字符的路径，比如/user/a，/user/aaa等  |
| /user/** | 可以匹配/user/*包含的内容之外，还可以匹配/user/a/b等多级目录形式 |

- **正则表达式路由映射**

有时为了兼容不同版本的客户端程序，后端系统需要创建不同版本的微服务来应对，比如：`user-v1`，`user-v2`等。默认情况下，`Zuul`自动为服务创建的路由表达式会采用服务名作为前缀，比如针对上面的`user-v1`和`user-v2`会产生`/user-v1`和`/user-v2`两个路径表达式来映射，这样生成出来的表示式规则单一，不利于管理。通常的做法是以版本号作为路由前缀，比如`/v1/userservice/`。这种使用版本号为前缀的`url`路径，可以很方便地对服务进行版本归类和管理。

我们可以使用`Zuul`来自定义服务与路由映射关系，创建类似于`/v1/userserivce/**`的路由规则。代码如下：

```java
@Bean
public PatternServiceRouteMapper serviceRouteMapper(){
    // 调用构造函数PatternServiceRouteMapper(String servicePattern, String routePattern)
  	// servicePattern 指定微服务的正则
  	// routePattern 指定路由正则
	return new PatternServiceRouteMapper("(?<name>^.+)-(?<version>v.+$)","${version}/${name}");
}
```

`PatternServiceRouteMapper`对象可以让开发者通过正则表达式自定义服务与路由映射的关系。构造函数第一个参数用来匹配服务名称的正则表达式，第二个参数根据服务名内容转换出的表达式规则。当定义了`PatternServiceRouteMapper`之后，只要符合第一个参数定义规则的服务名，就会优先使用该实现构建表达式，如果没有匹配上还是会使用默认的路由映射规则，即采用完整服务名作为前缀的路径表达式。

- **Zuul + Ribbon + Hystrix 超时熔断**

`Ribbon`配置：

1）当使用了`Eureka`注册中心，`zuul.routes`配置使用`service-id`的时候，通过`ribbon.ReadTimeout`和`ribbon.SocketTimeout`配置；

2）当`zuul.routes`配置使用`url`的时候,通过`zuul.host.connect-timeout-millis`和`zuul.host.socket-timeout-millis`配置；

如果需要对指定服务进行特殊配置，方式如下：

```properties
<serviceName>.ribbon.ReadTimeout  #serviceName为服务名
```

`Hystrix`配置：

如果`Zuul`配置了熔断`Fallback`的话，熔断超时也需要配置，方式如下：

```properties
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=30000
```

`default`代表默认，如果需要为某个指定服务特殊配置熔断超时策略，方式如下：

```properties
hystrix.command.<serviceName>.execution.isolation.thread.timeoutInMilliseconds=30000
```

如果想关闭`Hystrix`的重试机制可以通过下面的配置：

关闭全局重试机制：

```yml
zuul:
  retryable: false
```

关闭某个服务的重试机制：

```yml
zuul:
  routes:
    land-user:
      retryable: false
```

- **Header设置**

1）敏感Header设置

同一个系统中各个服务之间通过`Headers`来共享信息是没啥问题的，但是如果不想`Headers`中的一些敏感信息随着`HTTP`转发泄露出去话，需要在路由配置中指定一个忽略`Header`的清单。默认情况下，`Zuul`在请求路由时，会过滤`HTTP`请求头信息中的一些敏感信息，默认的敏感头信息通过`zuul.sensitiveHeaders`定义，包括`Cookie`、`Set-Cookie`、`Authorization`。配置的`sensitiveHeaders`可以用逗号分割。对指定路由的可以用下面进行配置:

```properties
# 对指定路由开启自定义敏感头
zuul.routes.[route].customSensitiveHeaders=true 
zuul.routes.[route].sensitiveHeaders=[这里设置要过滤的敏感头]
```

设置全局:

```properties
zuul.sensitiveHeaders=[设置要过滤的敏感头]
```

2）忽略Header设置

如果每一个路由都需要配置一些额外的敏感`Header`时，可以通过`zuul.ignoredHeaders`来统一设置需要忽略的`Header`。如:

```java
zuul.ignoredHeaders=[这里设置要忽略的Header]
```

在默认情况下是没有这个配置的，如果项目中引入了`Spring Security`，那么`Spring Security`会自动加上这个配置，默认值为: `Pragma, Cache-Control, X-Frame-Options, X-Content-Type-Options, X-XSS-Protection, Expries`。

此时，如果还需要使用下游微服务的`Spring Security`的`Header`时，可以增加下面的设置:

```properties
zuul.ignoreSecurityHeaders=false
```

- **熔断处理fallback**

当`Zuul`进行路由分发时，如果微服务没有启动，或者调用超时，`Zuul`可以使用一种降级功能，而不是将异常直接暴露出来。默认情况下，经过`Zuul`的请求都会使用`Hystrix`进行包裹，所以`Zuul`本身就具有断路器的功能`Zuul`，需要实现`ZuulFallbackProvider`接口。代码如下：

```java
@Component
public class HystrixFallbackConfig implements FallbackProvider {


    @Override
    public String getRoute() {
        //微服务配了路由的话，就用配置的名称
        //return "land-user";
        //*表示为所有微服务提供回退
        return "*";
    }

    @Override
    public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
        if (cause instanceof HystrixTimeoutException) {
            return response(HttpStatus.GATEWAY_TIMEOUT);
        } else {
            return this.fallbackResponse();
        }
    }

    public ClientHttpResponse fallbackResponse() {
        return this.response(HttpStatus.INTERNAL_SERVER_ERROR);
    }

    private ClientHttpResponse response(final HttpStatus status) {
        return new ClientHttpResponse() {

            @Override
            public HttpStatus getStatusCode() throws IOException {
                return status;
            }

            @Override
            public int getRawStatusCode() throws IOException {
                return status.value();
            }

            @Override
            public String getStatusText() throws IOException {
                return status.getReasonPhrase();
            }

            @Override
            public void close() {
            }

            @Override
            public InputStream getBody() throws IOException {
                return new ByteArrayInputStream("服务不可用，请稍后再试。".getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                // headers设定
                HttpHeaders headers = new HttpHeaders();
                // MediaType mt = new MediaType("application", "json", Charset.forName("UTF-8"));
                // headers.setContentType(mt);
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }

        };
    }

}
```

当`user`服务不可用时，访问返回结果如下：

![3.png](https://i.loli.net/2021/03/20/QwRXJf1lFWetrOy.png)

## 请求过滤

如果需要通过网关实现一些诸如过滤，权限验证等功能，就需要用到`Zuul`的请求过滤的功能。`ZuulFilter`类似于一个拦截器，会把请求拦截下来，然后做相应处理，最后决定是否放行。

`Zuul`大部分功能都是通过过滤器来实现的。`Zuul`中定义了四种标准过滤器类型，这些过滤器类型对应于请求的典型生命周期：

- **PRE**：这种过滤器在请求被路由之前调用。可以利用这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等。
- **ROUTING**：这种过滤器将请求路由到微服务。用于构建发送给微服务的请求，并使用`Apache HttpClient`或`Netflix Ribbon`构建和发送原始HTTP请求的位置。
- **POST**：请求在路由到微服务之后执行。示例包括向响应添加标准`Http`标头、收集统计信息和指标、以及将响应从源传输到客户端。
- **ERROR**：过滤器在其中一个阶段发生错误时执行。

除了默认的过滤器类型，`Zuul`还允许我们创建自定义过滤器类型。例如，自定义一个`Static`类型的过滤器，它直接在`Zuul`中生成响应，而不是将请求转发到后端的微服务。`Zuul`请求的生命周期如下图所示，详细描述了各种类型的过滤器执行顺序。

![1.png](https://i.loli.net/2021/03/20/ztoJUHyAuTO5ibW.png)

现在使用`Zuul`过滤器模拟一个对请求进行权限认证的功能，首先自定义一个过滤器并继承`ZuulFilter`，代码如下：

```java
@Configuration
public class ZuulFilterConfig extends ZuulFilter {
    
    // 过滤器类型，pre表示在请求路由之前进行拦截
    @Override
    public String filterType() {
        return "pre";
    }

    // 过滤器的执行顺序。当请求在一个阶段的时候存在多个过滤器时，需要根据该方法的返回值依次执行
    @Override
    public int filterOrder() {
        // 值越小执行顺行越靠前
        return 0;
    }

    // 确定过滤器是否生效
    @Override
    public boolean shouldFilter() {
        // 默认此类过滤器时false，不开启的，需要改为true
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        String token = request.getParameter("access-token");
        if (token == null || "".equals(token.trim()) {
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(HttpStatus.UNAUTHORIZED.value());
            ctx.addZuulResponseHeader(("content-type"), "text/html;charset=utf-8");
            ctx.setResponseBody("拒绝访问");
            return null;
        }
        
        // 否则正常执行业务逻辑.....
        return null;
    }
}
```

运行`Zuul`，不提供`access-token`参数访问服务，结果如下：

![2.png](https://i.loli.net/2021/03/20/tUwpLsmnYj6khyX.png)

##### 方法说明：

- `filterType()`：返回值为过滤器的类型，过滤器的类型决定了过滤器在哪个生命周期执行，上面代码中返回的是`pre`，表示是在路由之前执行过滤，其他可选值还有`post`，`error`，`route`和`static`，也可以自定义。
- `filterOrder()`：返回过滤器的执行顺序，当有多个过滤器时，这个方法定义执行顺序。
- `shouldFilter()`：这个方法用来判断这个过滤器是否执行，`true`就是执行，在实际使用中可以根据当前请求地址来决定要不要执行这个过滤器，这里为了测试，直接返回`true`了。
- `run()`：过滤器的具体业务逻辑，上面例子中，有`access-token`参数的请求放行，否则拦截下来。首先需要设置`ctx.setSendZuulResponse(false);`表示这个请求就不进行路由了，然后再设置`http`状态码和具体返回的`body`内容。在实际项目中，在`run`方法中可能是先获取用户对象，然后做权限判断等，并根据具体的业务具体实现。`run`方法的返回值在目前的版本中没有任何意义，可随意处理。













