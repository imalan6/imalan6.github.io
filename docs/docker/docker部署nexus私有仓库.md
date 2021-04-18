# docker部署nexus私有仓库

## 简介

最近项目需要对接银行系统，对方提供了一些jar包，这些三方`jar`是没有上传到中央仓库的，所以无法直接在`maven`中依赖，因此决定搭建一个`Maven`私服来处理。`maven`仓库的使用结构如下图：

![0.png](https://i.loli.net/2021/04/17/hpIPe8K5FZV2bGr.png)

通常，我们开发项目并没有使用到虚线标识的那两部分，基本都是通过本机的`Maven`直接访问中央仓库，下载`jar`包到本地仓库。现在我们需要搭建中间虚线部分。

## docker安装nexus

##### 1、下载镜像

官方镜像地址：https://hub.docker.com/r/sonatype/nexus3/tags

```shell
#docker pull sonatype/nexus3 	//下载镜像
```

##### 2、安装

```shell
#mkdir /usr/local/docker/nexus //新建目录
#chmod 777 /usr/local/docker/nexus //修改权限

//nexus默认使用8081端口
#docker run -d --restart=always -p 8081:8081 --name nexus -v /usr/local/docker/nexus:/nexus-data sonatype/nexus3
```

##### 3、nexus密码

安装完成后可访问管理平台：http://ip:8081

默认管理员用户名：admin 	密码：admin123

如果提示密码不对，需要到容器里面去修改密码。方式如下：

```shell
//进入容器
#docker exec -it nexus /bin/bash

//进入/opt/sonatype/sonatype-work/sonatype-work/目录，找到admin.password文件，里面的内容就是密码
#cat admin.password
```

![1.png](https://i.loli.net/2021/04/17/3SrTRk96DUbB8JH.png)

红色部分就是`admin`的密码。这个只是临时密码，修改密码后`admin.password`文件会消失。

## 配置nexus

登录`nexus`管理平台后（注意必须`admin`登录才行，不然只能浏览模式），可以看到如下界面：

![2.png](https://i.loli.net/2021/04/17/P1VKCN23LlxHecZ.png)

##### 1、创建Blob stores

在创建`repository`之前，还需要先指定文件存储目录，便于统一管理。就需要创建`Blob stores`

![3.png](https://i.loli.net/2021/04/17/Y2nwk3VOENtD1gJ.png)

创建好后可以看到`blob stores`有两个，一个是系统默认的，一个是刚创建的。如果不想自己创建，使用系统默认的文件存储目录（在`sonatype-work/nexus3/blobs`）也是可以的。到时候创建`repository`时，存储目录选择`default`就可以了。

![4.png](https://i.loli.net/2021/04/17/csqLzlXiMZPHnoY.png)

##### 2、nexus仓库

![5.png](https://i.loli.net/2021/04/17/kZXUbVucpWAL4rz.png)

如图所示，代理仓库负责代理远程中央仓库，托管仓库负责本地资源，组资源库 = 代理资源库 + 托管资源库。

##### 3、创建proxy repository代理仓库

![6.png](https://i.loli.net/2021/04/17/zUITYHaG8CN5w71.png)

选择`maven2(proxy)`，代理仓库

![7.png](https://i.loli.net/2021/04/17/Im19dukNKWTBGsz.png)

设置代理仓库

![8.png](https://i.loli.net/2021/04/17/FjS8pawX4EyRMbv.png)

 

![9.png](https://i.loli.net/2021/04/17/O3pVLri8bX7HuQz.png)

其他的可以采用默认，以后需要修改的可以再修改。

##### 4.创建hosted repository仓库

![10.png](https://i.loli.net/2021/04/17/HKU7ros4lLBx56F.png)

 

 ![11.png](https://i.loli.net/2021/04/17/UINcZi5HoxdJjpm.png)

上图的Hosted设置选项，选项中有三个值：

**Allow redeploy**：允许同一个版本号下重复提交代码, `nexus`以时间区分

**Disable redeploy**：不允许同一个版本号下重复提交代码

**Read-Only**：不允许提交任何版本

原生的`maven-releases`库是`Disable redeploy`设置， `maven-snapshots`是`Allow redeploy`。

##### 5、创建group repository组仓库

![12.png](https://i.loli.net/2021/04/17/v4pfBeuncTIWoqJ.png)



![13.png](https://i.loli.net/2021/04/17/Oz7GkZhesqTaCUQ.png)

将`hosted repositories`宿主仓库的顺序放在`proxy repositories`代理仓库之前，因为一个`group`仓库组中可以包括宿主仓库和代理仓库。而整个`group repository`是作为一个`public repository`给用户使用的。

所以当查找`jar`包的时候，如果代理资源库在前面，那就是先从远程去查找jar包，而不是先从宿主仓库（本地仓库）去查找jar包。

## 设置maven

`Maven`下的`setting.xml`文件和项目中的`pom.xml`文件的关系是：`settting.xml`文件是全局设置，而`pom.xml`文件是局部设置。`pom.xml`文件对于项目来说，是优先使用的。而`pom.xml`文件中如果没有配置镜像地址的话，就按照`settting.xml`中定义的地址去查找。

##### 1、修改maven配置文件setting.xml

![14.png](https://i.loli.net/2021/04/17/km1wdMv2HolNqPS.png)

如上图方式获取组仓库`smart_group`的仓库地址，修改`setting.xml`文件如下：

```xml
  <!--nexus服务器,id为组仓库name-->
  <servers>  
    <server>  
        <id>smart_group</id>  
        <username>admin</username>  
        <password>admin123</password>  
    </server>   
  </servers>  
  <!--仓库组的url地址，id和name可以写组仓库name，mirrorOf的值设置为central-->  
  <mirrors>     
    <mirror>  
        <id>smart_group</id>  
        <name>smart_group</name>  
        <url>http://******:8081/repository/smart_group/</url>  
        <mirrorOf>central</mirrorOf>  
    </mirror>     
  </mirrors> 
```

修改后可以重新编译项目，必须添加参数`-U`,（`-U，--update-snapshots`，强制更新`releases`、`snapshots`类型的插件或依赖库，否则`maven`一天只会更新一次`snapshot`依赖）。代理仓库会从远程中央仓库下载`jar`包

```shell
#mvn clean compile -U
```

![15.png](https://i.loli.net/2021/04/17/rdEwaMizfIs96UT.png)

![16.png](https://i.loli.net/2021/04/17/6WV3FtawuoX9cP1.png)

这个时候可以看到代理仓库已经从中央仓库下载了项目编译需要的jar包。同样地，在组仓库中也能看到所有的`jar`包，包括代理仓库和宿主仓库的。

##### 2、通过管理平台上传三方jar包

有些`jar`是第三方提供的，在中央仓库中是没有的，我们可以上传这些本地三方jar包到`hosted repository`宿主仓库中。

![17.png](https://i.loli.net/2021/04/17/UGMry9X7tLiFlNj.png)

![18.png](https://i.loli.net/2021/04/17/GKkPDs3qUTh5unY.png)

上传成功后，就可以看到`hosted repository`和`group repository`中已经有了刚上传的三方`jar`包

![19.png](https://i.loli.net/2021/04/17/JyfujtZLvw3W7gc.png)

然后项目中通过编译打包就可以下载`jar`包到本地`repository`中了，记住`maven  clean`、`compile`、`package`首次执行时加参数`-U`。

##### 3、通过命令上传三方jar包

在`setting.xml`配置文件中添加`hosted repository server`

```xml
    <!--id自定义，但是在使用命令上传的时候会用到-->
    <server>  
        <id>smart_hosted</id>  
        <username>admin</username>  
        <password>admin123</password>  
     </server>
```

使用如下命令上传jar包：

```shell
mvn deploy:deploy-file -DgroupId=com.alan6.land -DartifactId=land-user -Dversion=0.0.1 -Dpackaging=jar -Dfile=d:\land-service-user.jar -Durl=http://ip:8081/repository/smart_group/ -DrepositoryId=smart_group
```

命令解释：

-DgroupId=com.alan6.land　　　　 　　　　　　　　　　　　自定义

-DartifactId=land-user　　　 　　　　　　　　　　　　　　　自定义

-Dversion=0.0.1　　　　　 　　　　　　　　　　　　　　　　自定义，三个自定义，构成`pom.xml`文件中的标识

-Dpackaging=jar　　　　　　　　　　　　　　　　　　　　传的类型是jar类型

-Dfile=D:\jar\land-user-0.0.1.jar　　　　　　　　　　　　　　`jar`包的本地磁盘位置

-Durl=http://ip:8081/repository/smart_hosted/　　　　        `hosted`资源库的地址

-DrepositoryId=smart_hosted　　　　　　　　　　　　　　　需要和`setting.xml`文件中配置的ID一致

![20.png](https://i.loli.net/2021/04/17/tZ3WCwloeHTxB7I.png)

上传成功后，`hosted repository`中已经可以看到了

![21.png](https://i.loli.net/2021/04/17/4Bgj5cKpueV7NxQ.png)

## deploy部署jar包到私服

##### 1、release和snapshots jar包区别

`SNAPSHOT`版本代表不稳定（快照版本），还在处于开发阶段，随时都会有变化。当上传同样的版本号`jar`包的时候，`SNAPSHOT`会在版本号的后面自动追加一串新的数字，即日志标签，`nexus`会根据日志标签区分出不同的版本，在`maven`引用时，如果使用的是`snapshot`版本，重新导入`maven`的时候，会去私库拉取最新上传的代码。

`RELEASE`则代表稳定的版本（发布版本），一般上线后都会改用`RELEASE`版本。也就是说1.0，2.0这样的版本只能有一个，也就是说当前版本号下，不可能出现不同的`jar`。

##### 2、创建snapshot仓库

可以在`nexus`上添加一个`snapshot`仓库，专门用于存放`snapshot`版本的jar包。`snapshot`仓库也是`hosted`类型的，创建方式和`hosted repository`类型。创建好后，加入到组仓库里面就可以了。

![22.png](https://i.loli.net/2021/04/17/DZC1haWrOiwSu76.png)

##### 3、项目pom.xml文件配置

可以在项目的`pom`文件中设置具体的`releases`库和`snapshots`库。如果当前项目的版本号的后缀名中带着`-SNAPSHOT`，类似`<version>***-SNAPSHOT</version>`，就会将项目`jar`包提交到`snapshots`库中，没有带`-SNAPSHOT`的话会提交`releases`库中。

在`pom.xml`文件中配置`distributionManagement`节点如下，在项目中执行`deploy`命令后，`jar`包将会被上传到`nexus`中。

```xml
    <distributionManagement>
        <repository>
            <id>nexus-releases</id> <!--release版本仓库-->
            <name>Nexus Release Repository</name>
            <url>http://ip:8081/repository/smart_hosted/</url>
        </repository>
        <snapshotRepository>
            <id>nexus-snapshots</id> <!--snapshot版本仓库-->
            <name>Nexus Snapshot Repository</name>
            <url>http://ip:8081/repository/smart_snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
```

默认地，`maven`编译打包不会下载`SNAPSHOT`版本的`jar`包，所以还需要在`pom.xml`文件中配置支持下载`snapshot`版本`jar`包。

```xml
    <repositories>
        <repository>
            <id>smart_group</id>
            <url>http://ip:8081/repository/smart_group/</url>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>
```

至此，`nexus`搭建完毕，支持本地部署依赖`jar`包