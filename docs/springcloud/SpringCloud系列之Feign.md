# SpringCloud系列之Feign

## 简介

`Feign`是一个声明式、模板化的`HTTP`客户端，简化了系统发起`HTTP`请求。创建它时，只需要创建一个接口，然后加上`@FeignClient`注解就行了。使用它调用其他服务时，就像调用本地方法一样，完全感知不到这是在调用远程的方法，也感知不到背后发起的`HTTP`请求。`Feign`默认集成了`Ribbon`，并和`Eureka`结合，实现了负载均衡的效果。`Feign`功能如下：

- `Feign`采用的是基于接口的注解；
- 支持HTTP请求和响应的压缩；
- `Feign`整合了`Ribbon`，具有负载均衡的能力；
- 整合了`Hystrix`，具有熔断的能力。

## 使用方法

1）`pom.xml`文件添加依赖：

```xml
 <!--添加Feign依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

2）配置启动类，添加`@EnableFeignClient`注解：

```java

@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients(basePackages = {"com.landcode.land.service.user.service"})
public class UserApplication {
    public static void main(String[] args) {
        // TODO Auto-generated method stub
        SpringApplication.run(ConsumerApplication.class, args);
    }
}
```

3）定义一个`Feign`接口，通过`@FeignClient`（“服务名”）来指定调用哪个服务。

```java
@FeignClient(value = "land-hi")
public interface ITestHi {
    @RequestMapping("/hi")
    public String hi();
}
```

上述`ITestHi`接口调用`land-hi`服务，`hi`方法调用`land-hi`服务的`/hi`接口。

## 自定义Feign配置类

`Feign`也支持自定义配置，如下配置自定义重试机制，错误处理，拦截器demo。

- **代码配置方式**

```java
@Configuration
class FeignConfig {
    
    //自定义重试机制
    @Bean
    public Retryer feignRetryer() {
        //fegin提供的默认实现，最大请求次数为5，初始间隔时间为100ms，下次间隔时间1.5倍递增，重试间最大间隔时间为1s，
        return new Retryer.Default();
    }

	//错误处理
    @Bean
    public ErrorDecoder feignError() {
        return (key, response) -> {
            if (response.status() == 400) {
                log.error("请求服务400参数错误,返回:{}", response.body());
            }

            if (response.status() == 404) {
                log.error("请求服务404异常,返回:{}", response.body());
            }

            // 其他异常交给Default去解码处理
            return new ErrorDecoder.Default().decode(key, response);
        };
    }

    /**
     * fegin 拦截器
     * @return
     */
    @Bean
    public RequestInterceptor cameraSign() {

        return template -> {
            // 如果是get请求
            if (template.method().equals(Request.HttpMethod.GET.name())) {
                //获取到get请求的参数
                Map<String, Collection<String>> queries = template.queries();
            }

            //如果是Post请求
            else if (template.method().equals(Request.HttpMethod.POST.name())) {
                //获得请求body
                String body = template.requestBody().asString();
                JSONPObject request = JSON.parseObject(body, JSONPObject.class);
            }

            //根据请求参数生成的签名
            String sign = "*******";
            
            //放入url之后
            template.query("sign", sign);
            
            //放入请求body中
            String newBody = body + sign;
            
            template.body(Request.Body.encoded(newBody.getBytes(StandardCharsets.UTF_8), StandardCharsets.UTF_8));
        };
    }
}
```

当然，`FeignConfig`也可以无须定义在Spring容器中，直接在`@FeignClient`注解上使用也可以生效。

```java
@FeignClient(value = "hi", configuration = FeignConfig.class)
```

- **属性配置方式**

从`Spring Cloud Edgware`开始，`Feign`支持使用属性自定义配置。对于一个指定名称的`FeignClient`（例如该`FeignClient`的名称为`feignName` ），`Feign`支持如下配置项：

```yaml
feign:
  client:
    config:
      feignName:
        connectTimeout: 5000  # 相当于Request.Options
        readTimeout: 5000     # 相当于Request.Options
        # 配置Feign的日志级别，相当于代码配置方式中的Logger
        loggerLevel: full
        # Feign的错误解码器，相当于代码配置方式中的ErrorDecoder
        errorDecoder: com.example.SimpleErrorDecoder
        # 配置重试，相当于代码配置方式中的Retryer
        retryer: com.example.SimpleRetryer
        # 配置拦截器，相当于代码配置方式中的RequestInterceptor
        requestInterceptors:
          - com.alanotes.FooRequestInterceptor
          - com.example.BarRequestInterceptor
        decode404: false
```

当然不建议配置`retryer`重试机制，Spring Cloud Camden以及之后的版本中，`Spring Cloud`关闭了`Feign`的重试，而是使用`Ribbon`的重试。如果自己再定义`Feign`的重试，可能会造成混乱。



