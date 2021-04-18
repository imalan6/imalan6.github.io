# Idea使用docker插件部署服务到远程服务器

## docker部署单个服务

**1、Idea安装docker插件**

首先给`Idea`安装`docker`插件，方式为：File ——> Settings ——> Plugins，安装后重启IDE

![0.png](https://i.loli.net/2021/04/17/bDxl6d7HcGWuYES.png)

**2、配置远程docker主机**

1）首先登陆远程`docker`主机，修改配置文件`/usr/lib/systemd/system/docker.service`

```shell
#vim /usr/lib/systemd/system/docker.service
```

打开文件，找到`ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock`这一行，在后面添加`-H tcp://0.0.0.0:2375`，表示打开2375端口，支持远程连接`docker`

![1.png](https://i.loli.net/2021/04/17/lIYB6kqX7EdQSzK.png)

2）修改文件后保存，重新加载配置文件并重启`docker`服务 

```shell
#systemctl daemon-reload
#systemctl restart docker.service
```

3）Idea配置docker主机，File ——> Settings ——>Build，Execution，Depolyment ——> Docker，添加一个`Docker`主机，`TCP socket`中添加远程主机+端口，以`tcp://`开头，`tcp://192.168.0.6:2375`，添加后会自动连接远程`docker`主机，下方看到`connection successful`表示连接成功，否则表示连接失败。

 ![2.png](https://i.loli.net/2021/04/17/POVNJFCK9Ihj8ae.png)

4）`docker`主机连接成功后，在`docker`插件面板中可以看到`docker`主机的容器和镜像，以及`docker`容器运行的日志等信息

![3.png](https://i.loli.net/2021/04/17/Ggja1QRTyAz5d6I.png)

**3、创建Dockerfile文件**

在服务根目录创建`docker`文件夹，路径如下

![4.png](https://i.loli.net/2021/04/17/dCVfuEjQaFbY6MI.png)

在`docker`目录创建一个`Dockerfile`文件，内容：

```dockerfile
FROM java:8
VOLUME /tmp
ADD land-service-hi.jar app.jar
EXPOSE 8800
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

内容参数解释见docker

**4、配置pom文件**

在`build`的`plugins`插件标签中添加：

```xml
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <!-- 打包成可执行jar包 -->
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <encoding>utf-8</encoding>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>1.0.0</version>
                <!--将插件绑定在某个phase执行-->
                <executions>
                    <execution>
                        <id>build-image</id>
                        <!--用户只需执行mvn package ，就会自动执行mvn docker:build-->
                        <phase>package</phase>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <!--指定docker文件目录-->
                    <dockerDirectory>${project.basedir}/docker</dockerDirectory>
                    <!--指定生成的镜像名-->
                    <imageName>land/${project.artifactId}</imageName>
                    <!--指定标签-->
                    <imageTags>
                        <imageTag>latest</imageTag>
                    </imageTags>
                    <!--指定远程 docker api地址-->
                    <dockerHost>http://192.168.0.6:2375</dockerHost>
                    <!-- 这里是复制 jar 包到 docker 容器指定目录配置 -->
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <!--jar 包所在的路径  此处配置的 即对应 target 目录-->
                            <directory>${project.build.directory}</directory>
                            <!-- 需要包含的 jar包 ，这里对应的是 Dockerfile中添加的文件名　-->
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
        </plugins>
```

**5、打包部署**

通过上面一系列配置后，`maven`打包服务时将自动build 镜像到远程主机，通过`docker`插件的面板可以看到镜像已经推送到远程主机

```
#mvn clean package -DskipTests
```

![5.png](https://i.loli.net/2021/04/17/hDml8kZTunFQJow.png)

## docker部署多个服务

**1、服务创建容器**

给每个服务都进行上面的配置后，使用`maven`在父项目打包时，将自动上传所有服务的镜像到远程`docker`主机。这样，虽然上传了服务镜像到远程主机，但远程主机并没有创建和启动对应的容器。

方法：可以通过在`docker`面板手动创建容器，以后每次使用`maven`打包服务时，不仅会上传镜像同时还会自动重启远程主机的`docker`容器。

![6.png](https://i.loli.net/2021/04/17/4FVKcq8twRahd5n.png)

同样地，为每个服务手动创建一个容器，然后在项目根目录打包时，每个服务的镜像都会上传到远程`docker`主机并自动重启容器。

**2、使用docker-compose**

可以使用上面手动创建容器的方法，让远程`docker`主机启动容器。也可以待各个服务镜像上传到远程主机后，使用`docker-compose`完成服务编排，然后启动各个微服务。

1）在远程`docker`主机编辑`docker-compose.yml`文件

```yaml
version: '2'
services:
  land-eureka:
    image: land/land-eureka:latest
    container_name: land-eureka
    ports:
      - '8761:8761'

  land-zuul:
    image: land/land-zuul:latest
    container_name: land-zuul
    ports:
      - '9000:9000'

  land-netty:
    image: land/land-netty:latest
    container_name: land-netty
    ports:
      - '8000:8000'

  land-service-hi:
    image: land/land-service-hi:latest
    container_name: land-service-hi
    ports:
      - '8800:8800'

  land-service-consumer:
    image: land/land-service-consumer:latest
    container_name: land-service-consumer
    ports:
      - '8801:8801'
```

2）在`docker-compose.yml`文件目录下运行`docker-compose`命令，启动容器

```shell
#docker-compose up -d
```

![7.png](https://i.loli.net/2021/04/17/nKRpIxDeHWrmCFP.png)

 