# docker部署rabbitmq和rabbitmq集群

## 部署RabbitMQ单机

##### 1、查询rabbitmq镜像

```shell
#docker search rabbitmq:management
```

##### 2、拉取rabbitmq镜像

```shell
#docker pull rabbitmq:management
```

##### 3、创建并启动容器

- 创建和启动

```shell
#docker run -d --hostname my-rabbit --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:management
```

- 创建和启动（同时设置用户和密码） 

```shell
#docker run -d --hostname my-rabbit --name rabbitmq --restart always -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin -v /etc/localtime:/etc/localtime:ro -v /usr/local/rabbitmq/data:/var/lib/rabbitmq -p 15672:15672 -p 5672:5672 -p 25672:25672 -p 61613:61613 -p 1883:1883 rabbitmq:management
```

说明：

--hostname：指定容器主机名称

--name：指定容器名称

-p：将mq端口号映射到本地，15672为控制台端口号（用于管理`rabbitmq`），5672为应用访问端口号（应用程序访问）。

##### 4、查看rabbitmq日志，检查运行状况

```shell
#docker logs rabbit
```

## 部署RabbitMQ集群

### RabbitMQ集群概念

`RabbitMQ`有三种模式：

- 单机模式

- 普通集群模式

- 镜像集群模式

单机模式单独运行一个`rabbitmq`实例，而集群模式需要创建多个`rabbitmq`实例。

#### 普通集群模式

**概念：**

默认的集群模式。需要创建多个`RabbitMQ`节点。但对于`Queue`和消息来说，只存在于其中一个节点，其他节点仅同步元数据，即队列的结构信息。

当消息进入`Queue`后，如果`Consumer`从创建Queue的这个节点消费消息时，可以直接取出来；但如果`consumer`连接的是其他节点，那`rabbitmq`会把`queue`中的消息从创建它的节点中取出并经过连接节点转发后再发送给`consumer`。

所以`consumer`应尽量连接每一个节点。并针对同一个逻辑队列，要在多个节点建立物理`Queue`。否则无论`consumer`连接哪个节点，都会从创建`queue`的节点获取消息，会产生瓶颈。

![0.png](https://i.loli.net/2021/04/16/Andp5TIB43vo91O.png)

**特点：**

1）`Exchange`的元数据信息在所有节点上是一致的，而`Queue`（存放消息的队列）的完整数据则只会存在于创建它的那个节点上。其他节点只知道这个`queue`的 metadata信息和一个指向`queue`的`owner node`的指针；

2）`RabbitMQ`集群会始终同步四种类型的内部元数据（类似索引）：

- 队列元数据：队列名称和它的属性

- 交换器元数据：交换器名称、类型和属性

- 绑定元数据：一张简单的表格展示了如何将消息路由到队列

- `vhost`元数据：为`vhost`内的队列、交换器和绑定提供命名空间和安全属性

因此，当用户访问其中任何一个`RabbitMQ`节点时，通过`rabbitmqctl`查询到的元数据信息都是相同的。

3）无法实现高可用性，当创建`queue`的节点故障后，其他节点是无法取到消息实体的。如果做了消息持久化，那么得等创建`queue`的节点恢复后，才可以被消费。如果没有持久化的话，就会产生消息丢失的现象。

#### 镜像集群模式

**概念：**

把队列做成镜像队列，让各队列存在于多个节点中，属于`RabbitMQ`的高可用性方案。镜像模式和普通模式的不同在于，`queue`和`message`会在集群各节点之间同步，而不是在`consumer`获取数据时临时拉取。

**特点：**

1）实现了高可用性。部分节点挂掉后，不会影响`rabbitmq`的使用

2）降低了系统性能。镜像队列数量过多，大量的消息同步也会加大网络带宽开销

3）适合对可用性要求较高的业务场景

### RabbitMQ普通集群部署

##### 1、拉镜像

```shell
#docker pull rabbitmq:management
```

##### 2、运行容器

```shell
#docker run -d --hostname rabbit_host1 --name rabbitmq1 -p 15672:15672 -p 5672:5672 -e RABBITMQ_ERLANG_COOKIE='rabbitmq_cookie' rabbitmq:management

#docker run -d --hostname rabbit_host2 --name rabbitmq2 -p 5673:5672 --link rabbitmq1:rabbit_host1 -e RABBITMQ_ERLANG_COOKIE='rabbitmq_cookie' rabbitmq:management

#docker run -d --hostname rabbit_host3 --name rabbitmq3 -p 5674:5672 --link rabbitmq1:rabbit_host1 --link rabbitmq2:rabbit_host2 -e RABBITMQ_ERLANG_COOKIE='rabbitmq_cookie' rabbitmq:management
```

主要参数：

- -p 15672:15672	`management`界面管理访问端口

- -p 5672:5672    `amqp`访问端口

- --link   容器之间连接

`Erlang Cookie`值必须相同，也就是一个集群内`RABBITMQ_ERLANG_COOKIE`参数的值必须相同。因为`RabbitMQ`是用`Erlang`实现的，`Erlang Cookie`相当于不同节点之间通讯的密钥，`Erlang`节点通过交换`Erlang Cookie`获得认证。

##### 3、加入节点到集群

设置节点1：

```shell
#docker exec -it myrabbit1 bash
#rabbitmqctl stop_app
#rabbitmqctl reset
#rabbitmqctl start_app
#exit
```

设置节点2，加入到集群：

```shell
#docker exec -it myrabbit2 bash
#rabbitmqctl stop_app
#rabbitmqctl reset
#rabbitmqctl join_cluster --ram rabbit@rabbitmq_host1
#rabbitmqctl start_app
#exit
```

设置节点3，加入到集群：

```shell
#docker exec -it myrabbit3 bash
#rabbitmqctl stop_app
#rabbitmqctl reset
#rabbitmqctl join_cluster --ram rabbit@rabbitmq_host1
#rabbitmqctl start_app
#exit
```

主要参数：

- --ram   表示设置为内存节点，忽略次参数默认为磁盘节点。该配置启动了3个节点，1个磁盘节点和2个内存节点。

设置好之后，使用http://ip:15672进行访问，默认账号密码：`guest/guest`

![1.png](https://i.loli.net/2021/04/16/sV3yDXfNcaST18k.png)

 可以看到，已经有多个节点了。

### RabbitMQ镜像集群部署

##### 1、策略policy概念

使用`RabbitMQ`镜像功能，需要基于`RabbitMQ`策略来实现，策略`policy`是用来控制和修改群集范围的某个`vhost`队列行为和`Exchange`行为。策略`policy`就是要设置哪些`Exchange`或者`queue`的数据需要复制、同步，以及如何复制同步。

为了使队列成为镜像队列，需要创建一个策略来匹配队列，设置策略有两个键“`ha-mode`和`ha-params`（可选）”。`ha-params`根据`ha-mode`设置不同的值，下表说明这些`key`的选项。

![2.png](https://i.loli.net/2021/04/16/gwd83cNSI7h1orZ.png)

##### 2、添加策略

登录rabbitmq管理页面 ——> Admin ——> Policies ——> Add / update a policy

![3.png](https://i.loli.net/2021/04/16/jV7H18gLonJaGDu.png)

- name：随便取，策略名称

- Pattern：`^`匹配符，只有一个`^`代表匹配所有

- Definition：`ha-mode=all`为匹配类型，分为3种模式：`all`（表示所有的`queue`）

或者使用命令：

```shell
#rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```

##### 3、查看效果

此策略会同步所在同一`VHost`中的交换器和队列数据。设置好`policy`之后，使用http://ip:15672再次进行访问，可以看到队列镜像同步。

![4.png](https://i.loli.net/2021/04/16/hazUZvgj9NCsobG.png)

## Springboot配置RabbitMQ集群

##### 1、配置RabbitMQ单机

```yaml
spring:
　　rabbitmq:
　　　　host: localhost
　　　　port: 5672
　　　　username: username
　　　　password: password
```

或者使用`addresses`

```yaml
spring:
　　rabbitmq:
　　　　addresses:ip1:port1
　　　　username: username
　　　　password: password
```

##### 2、配置RabbitMQ集群

`addresses`节点用逗号分隔

```yaml
spring:
　　rabbitmq:
　　　　addresses:ip1:port1,ip2:port2,ip3:port3
　　　　username: username
　　　　password: password
```