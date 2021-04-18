# Nginx反向代理集群部署

## 概述

#### Nginx

`Nginx`是一款非常优秀的反向代理工具，支持请求分发，负载均衡，以及缓存等。在请求处理上，`Nginx`采用`epoll`模型，这是一种基于事件监听的模型，因而其具备非常高效的请求处理效率，单机并发能力能够达到百万。`Nginx`接收到的请求可以通过负载均衡策略分发到其下一级应用服务器。对于一些特大型的网站，单机的`Nginx`并发能力是有限的，而`Nginx`本身并不支持集群模式，因而对`Nginx`的横向扩展显得尤为重要。

#### Keepalived

`Keepalived`是一款服务器状态检测和故障切换的工具。在其配置文件中，可以配置主备服务器和该服务器的状态检测请求。`Keepalived`起初是专为`LVS`负载均衡软件设计的，用来管理并监控`LVS`集群系统中各个服务节点的状态，后来又加入了可以实现高可用的`VRRP`功能。因此，`Keepalived`除了能够管理`LVS`软件外，还可以用于解决其他服务的高可用解决方案。

`Keepalived`主要是通过`VRRP`协议实现高可用功能的。`VRRP`是`Virtual Router Redundancy Protocol`（虚拟路由冗余协议）的缩写，`VRRP`出现的目的就是为了解决静态路由的单点故障问题，它能保证当个别节点宕机时，整个网络可以不间断地运行。所以，`Keepalived`一方面具有配置管理`LVS`的功能，同时还具有对`LVS`下面节点进行健康检查的功能，另一方面也可以实现系统网络服务的高可用功能。

`Keepalived`的高可用服务的故障切换转移功能，是通过`VRRP`来实现的。在`Keepalived`服务工作时，主`Master`节点会不断地向备节点发送（多播的方式）心跳消息，用来告诉备`Backup`节点自己还活着。当主节点发生故障时，就无法发送心跳的消息了，备节点也因此无法继续检测到来自主节点的心跳了。于是就会调用自身的接管程序，接管主节点的IP资源和服务。当主节点恢复时，备节点又会释放主节点故障时自身接管的IP资源和服务，恢复到原来的备用角色。

## 结构图

![0.png](https://i.loli.net/2021/04/11/x2mdPGtHRSofLQw.png)

## 集群部署

#### 安装Keepalived+Nginx

选择两台服务器，这里使用的是`centos7`，安装`Keepalived`和`Nginx`。

安装`Keepalived`：

```bash
yum -y install keepalived
```

运行`Keepalived`：

```shell
systemctl start keepalived
```

安装`Nginx`：

```sh
yum -y install nginx
```

运行`Nginx`：

```shell
systemctl start nginx
```

#### 配置文件

Master服务器配置文件：

```shell
! Configuration File for Keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   #vrrp_strict #此处不注释掉，无法ping通VIP
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

#监控Nginx进程状态的脚本
vrrp_script check_nginx {
    script "/usr/local/server/check_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER   #master为master，backup则为backup
    interface ens33   #修改为实际的网卡
    virtual_router_id 51   #此处MASTER和BACKUP必须一致，但是和局域网中其他的keeplived集群不能相同
    priority 100  #数字越大，优先级越高。master高于其他
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    track_script { #执行监控Nginx进程脚本
		check_nginx
    }
    virtual_ipaddress {
        192.168.0.8 #虚拟IP地址，可以配置多个
    }
}

```

Backup服务器配置文件(仅列不同项)：

```shell
state BACKUP  #master为master，backup则为backup 
priority 80  #数字越大，优先级越高。master高于其他
```

`Keepalived`配置文件中加了注释符号#的配置项需要修改，其余的可以不变。主要修改项如下：

- `router_id` 是路由标识，在一个局域网里面应该是唯一的

- `vrrp_instance VI_1{...}`是一个`VRRP`实例，里面定义了`Keepalived`的主备状态、接口、优先级、认证和IP信息

- `state` 定义了VRRP的角色

- `interface`定义使用的接口，这里的网卡都`ens33`，根据实际填写

- `virtual_router_id`是虚拟路由ID标识，一组的`Keepalived`配置中主备都是一致的

- `priority`是优先级，数字越大，优先级越高

- `auth_type`是认证方式

- `auth_pass`是认证的密码

- `virtual_ipaddress｛...｝`定义虚拟IP地址，可以配置多个IP地址，这里定义为`192.168.0.8`

#### 脚本文件

注意，上面配置文件中的脚本配置

```shell
#监控Nginx进程状态的脚本
vrrp_script check_nginx {
    script "/usr/local/server/check_nginx.sh"
    interval 2
    weight 2
}
```

其中的脚本文件"`/usr/local/server/check_nginx.sh`"，用于检查`Nginx`状态，内容如下：

```sh
#!/bin/sh

nginxpid=$(ps -C nginx --no-header|wc -l)
#判断Nginx是否存活，如果不存活则尝试启动Nginx
if [ $nginxpid -eq 0 ];then
    systemctl start nginx
    sleep 2
    #等待2秒后，再次查看Nginx是否启动
    nginxpid=$(ps -C nginx --no-header|wc -l) 
    #如果Nginx还是没启动，停止Keepalived，让地址漂移到其他Nginx
    if [ $nginxpid -eq 0 ];then
        systemctl stop keepalived
   fi
fi
```

作用是检查`Nginx`进程是否存活，如果不存活就启动`Nginx`，并且2秒后再次检查，如果还是不存活，说明启动`Nginx`失败，然后关闭`Keepalived`服务，把虚拟地址转移给其他`Nginx`服务(`backup`)。

但是需要注意：在使用时，发现有一台服务器的`Nginx`服务启动后有4个`Nginx`进程，而`Nginx`服务关闭后，仍然存在2个`Nginx`进程，另外一台是正常的，可能是安装是出现问题。

`Nginx`服务启动后：

```shell
[root@192 ~]# ps aux | grep nginx
root      22177  0.0  0.0  10648  3392 ?        Ss   14:00   0:00 nginx: master process nginx -g daemon off;
101       22230  0.0  0.0  11088  1484 ?        S    14:00   0:00 nginx: worker process
root      22365  0.0  0.0 105496  1976 ?        Ss   14:02   0:00 nginx: master process /usr/sbin/nginx
Nginx     22366  0.0  0.0 108048  3376 ?        S    14:02   0:00 nginx: worker process
```

`Nginx`服务关闭后：

```sh
[root@192 ~]# ps aux | grep nginx
root      22177  0.0  0.0  10648  3392 ?        Ss   14:00   0:00 nginx: master process nginx -g daemon off;
101       22230  0.0  0.0  11088  1484 ?        S    14:00   0:00 nginx: worker process
```

所以针对这台服务器，使用上面的`Nginx`状态检查脚本，就无法正常检测出`Nginx`是否存活。因为当关闭`Nginx`服务后，脚本中的 "`ps -C Nginx --no-header|wc -l`" 语句仍然返回2，而不是0，也就不会执行后面启动`Nginx`的命令了。这时，就需要修改脚本，保证能正常检测`Nginx`是否存活。

## 测试

修改`Nginx`服务器的`html`主页文件"`/usr/share/Nginx/html/index.html`"，在`master`和`backup`上分别加入区分描述，用于测试使用。

分别启动`master`和`backup`服务器上的`Nginx`和`Keepalived`服务，浏览器访问http://192.168.0.8，返回如下：

![1.png](https://i.loli.net/2021/04/11/ME3ROILTktoVjxX.png)

然后关闭`master`服务器上的`Nginx`和`Keepalived`服务，返回如下：

![2.png](https://i.loli.net/2021/04/11/xPYh5Dy2uW4qLQ8.png)























