# docker部署redis

##### 1、搜索镜像

```shell
#docker search redis
```

##### 2、拉取镜像

```shell
#docker pull redis
```

##### 3、创建redis容器

```shell
#docker run -d --name redis --restart always -p 6379:6379 -v /usr/local/redis/data:/data redis --requirepass "123456" --appendonly yes 创建redis容器（指定配置文件）#docker run -d --name redis --restart always -p 6379:6379 -v /usr/local/redis/config:/etc/redis -v /usr/local/redis/data:/data redis redis-server /etc/redis/redis.conf --requirepass "123456" --appendonly yes
```

##### 参数说明：

- -p 6379:6379　　容器`redis`端口6379映射宿主主机6379

- --name redis　　容器名字为`redis`

- -v /usr/local/redis/conf:/etc/redis      `docker`镜像`redis`默认无配置文件，在宿主主机`/usr/local/redis/conf`下创建`redis.conf`配置文件，会将宿主机的配置文件复制到`docker`中

- -v /root/redis/redis01/data:/data　　容器`/data`映射到宿主机`/usr/local/redis/data`下

- -d redis 　　后台模式启动`redis`

- redis-server /etc/redis/redis.conf        `redis`将以`/etc/redis/redis.conf`为配置文件启动

- --appendonly yes　　开启`redis`的`AOF`持久化，默认为`false`，不持久化

