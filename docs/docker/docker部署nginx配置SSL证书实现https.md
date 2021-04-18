# docker部署nginx配置SSL证书实现https

## 拉镜像

```shell
#docker pull nginx
```

## 创建并启动容器

```shell
docker run -p 8443:443 --name nginx8443 \
-v /usr/local/docker/nginx8443/html:/usr/share/nginx/html \
-v /usr/local/docker/nginx8443/logs:/var/log/nginx \
-v /usr/local/docker/nginx8443/conf/nginx.conf:/etc/nginx/nginx.conf \
-v /usr/local/docker/nginx8443/conf/cert:/etc/nginx/cert \
-v /etc/localtime:/etc/localtime \
-d nginx
```

**volume映射参数：**

/usr/share/nginx/html：部署网站的根目录

/etc/nginx/nginx.conf：`nginx`配置文件

/etc/nginx/cert：证书存放目录

**说明**：因为服务器上的443端口已经被其他项目占用，这里使用8443端口来部署，记得打开防火墙端口限制。运行时会报如下错误：

![0.png](https://i.loli.net/2021/04/17/i2lUDGxr5PXzYvR.png)

这是由于`/etc/nginx/nginx.conf`是目录，却被映射到文件。只需要到`/usr/local/docker/nginx8443/conf/`目录下删除`nginx.conf`目录，然后新建一个`nginx.conf`配置文件即可，见第三步。

## 添加配置文件

到`/usr/local/docker/nginx8443/conf/`目录下删除`nginx.conf`目录，然后新建一个`nginx.conf`配置文件，内容如下：

```properties
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;
	
	ssl on;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;
 
    ssl_certificate /etc/nginx/cert/xxx.pem;     		#证书路径
    ssl_certificate_key /etc/nginx/cert/xxx.key; 		#请求认证 key 的路径

	server {
		listen	443;   #监听端口，ssl默认443端口。如果需要配置多个端口，可以继续添加server，用不同的端口就行
		server_name  www.xxx.com;   #服务器域名，需要和申请的证书匹配
		
		location / {
			root  /usr/share/nginx/html;  #网站根目录，和容器创建时指定的位置一致
			index index.html index.htm;
		}
	}
}
```

添加完配置文件后，重启`nginx`容器，并检查下日志看是否有`error`。

```shell
#docker restart nginx8443
#docker logs nginx8843
```

## 测试

重启容器，一切正常后，在`/usr/local/docker/nginx8443/html`目录下，新建一个`index.html`，输入hello world！，浏览器访问https://www.xxx.com:8443，即可正常访问。