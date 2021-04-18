# SpringCloud系列之Actuator

## 简介

微服务系统通常是分布式部署的，大部分服务都运行在不同的机器上，彼此通过服务接口交互调用，业务会经过多个微服务的处理和传递，如果期间出现了异常需要快速定位，就需要对微服务的状态进行实时监控。

`spring-boot-starter-actuator`，这个模块可以自动为`Spring Boot`创建的应用构建一系列的用于监控的端点。`Actuator`是`Spring Boot`提供的对应用系统的自省和监控的集成功能，可以查看应用配置的详细信息，例如自动化配置信息、创建的`Spring beans`以及一些环境属性等。使用`Actuator`可以很方便地对微服务状况进行监控。

## 监控端点

`Actuator`监控端点分为原生端点和用户自定义端点。自定义端点可以让用户自定义一些比较关心的指标，在运行期进行监控。而原生端点是由`actuator`提供的`web`接口，用来了解服务运行时的内部状况。原生端点可以分成三类：

- 应用配置类：用于查看服务在运行时的静态信息，比如自动配置信息，`Spring Bean`信息、配置文件信息、环境信息、请求映射等；
- 度量指标类：主要是服务运行时的动态信息，比如堆栈、请求链、健康指标、`metrics`信息等；
- 操作控制类：主要是指`shutdown`，用户可以发送请求关闭应用的监控功能。

`Actuator`提供了13个接口，如下表所示：

| HTTP方法 |      路径       |            描述            | 鉴权  |
| :------: | :-------------: | :------------------------: | :---: |
|   GET    |   /autoconfig   |   查看自动配置的使用情况   | true  |
|   GET    |  /configprops   | 查看配置属性，包括默认配置 | true  |
|   GET    |     /beans      |    查看bean及其关系列表    | true  |
|   GET    |      /dump      |         打印线程栈         | true  |
|   GET    |      /env       |      查看所有环境变量      | true  |
|   GET    |   /env/{name}   |       查看具体变量值       | true  |
|   GET    |     /health     |      查看应用健康指标      | false |
|   GET    |      /info      |        查看应用信息        | false |
|   GET    |    /mappings    |      查看所有url映射       | true  |
|   GET    |    /metrics     |      查看应用基本指标      | true  |
|   GET    | /metrics/{name} |        查看具体指标        | true  |
|   POST   |    /shutdown    |          关闭应用          | true  |
|   GET    |     /trace      |      查看基本追踪信息      | true  |

## 部署

1）加入`pom`依赖

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
</dependencies>
```

2）配置文件

```yaml
#启用监控
management:
  endpoints:
    web:
      exposure:
        include: "*"  #开放所有端点health，info，metrics，通过actuator/端点名，就可以获取对应的信息。如果不配置，默认只打开health和info端点
  endpoint:
    health:
      show-details: always  #未开启actuator/health时，我们获取到的信息是{"status":"UP"}，status的值还有可能是 DOWN。开启后打印详细信息
```

3）启动服务，输入 http://localhost:8081/actuator ，查看端点信息如下：

```json
{
    "_links":{
        "self":{
            "href":"http://127.0.0.1:8801/actuator",
            "templated":false
        },
        "archaius":{
            "href":"http://127.0.0.1:8801/actuator/archaius",
            "templated":false
        },
        "auditevents":{
            "href":"http://127.0.0.1:8801/actuator/auditevents",
            "templated":false
        },
        "beans":{
            "href":"http://127.0.0.1:8801/actuator/beans",
            "templated":false
        },
        "caches-cache":{
            "href":"http://127.0.0.1:8801/actuator/caches/{cache}",
            "templated":true
        },
        "caches":{
            "href":"http://127.0.0.1:8801/actuator/caches",
            "templated":false
        },
        "health":{
            "href":"http://127.0.0.1:8801/actuator/health",
            "templated":false
        },
        "health-component-instance":{
            "href":"http://127.0.0.1:8801/actuator/health/{component}/{instance}",
            "templated":true
        },
        "health-component":{
            "href":"http://127.0.0.1:8801/actuator/health/{component}",
            "templated":true
        },
        "conditions":{
            "href":"http://127.0.0.1:8801/actuator/conditions",
            "templated":false
        },
        "configprops":{
            "href":"http://127.0.0.1:8801/actuator/configprops",
            "templated":false
        },
        "env":{
            "href":"http://127.0.0.1:8801/actuator/env",
            "templated":false
        },
        "env-toMatch":{
            "href":"http://127.0.0.1:8801/actuator/env/{toMatch}",
            "templated":true
        },
        "info":{
            "href":"http://127.0.0.1:8801/actuator/info",
            "templated":false
        },
        "loggers":{
            "href":"http://127.0.0.1:8801/actuator/loggers",
            "templated":false
        },
        "loggers-name":{
            "href":"http://127.0.0.1:8801/actuator/loggers/{name}",
            "templated":true
        },
        "heapdump":{
            "href":"http://127.0.0.1:8801/actuator/heapdump",
            "templated":false
        },
        "threaddump":{
            "href":"http://127.0.0.1:8801/actuator/threaddump",
            "templated":false
        },
        "metrics-requiredMetricName":{
            "href":"http://127.0.0.1:8801/actuator/metrics/{requiredMetricName}",
            "templated":true
        },
        "metrics":{
            "href":"http://127.0.0.1:8801/actuator/metrics",
            "templated":false
        },
        "scheduledtasks":{
            "href":"http://127.0.0.1:8801/actuator/scheduledtasks",
            "templated":false
        },
        "httptrace":{
            "href":"http://127.0.0.1:8801/actuator/httptrace",
            "templated":false
        },
        "mappings":{
            "href":"http://127.0.0.1:8801/actuator/mappings",
            "templated":false
        },
        "refresh":{
            "href":"http://127.0.0.1:8801/actuator/refresh",
            "templated":false
        },
        "features":{
            "href":"http://127.0.0.1:8801/actuator/features",
            "templated":false
        },
        "service-registry":{
            "href":"http://127.0.0.1:8801/actuator/service-registry",
            "templated":false
        },
        "jolokia":{
            "href":"http://127.0.0.1:8801/actuator/jolokia",
            "templated":false
        },
        "hystrix.stream":{
            "href":"http://127.0.0.1:8801/actuator/hystrix.stream",
            "templated":false
        }
    }
}
```

4）输入http://127.0.0.1:8801/actuator/health，查看`health`端点信息如下：

```json
{
    "status":"UP",
    "details":{
        "diskSpace":{
            "status":"UP",
            "details":{
                "total":124945166336,
                "free":738893824,
                "threshold":10485760
            }
        },
        "refreshScope":{
            "status":"UP"
        },
        "discoveryComposite":{
            "status":"UP",
            "details":{
                "discoveryClient":{
                    "status":"UP",
                    "details":{
                        "services":[

                        ]
                    }
                },
                "eureka":{
                    "description":"Eureka discovery client has not yet successfully connected to a Eureka server",
                    "status":"UP",
                    "details":{
                        "applications":{

                        }
                    }
                }
            }
        },
        "hystrix":{
            "status":"UP"
        }
    }
}
```