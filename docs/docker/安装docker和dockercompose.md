# 安装docker，docker compose

## 安装docker

##### 1、升级所有包（这步版本够用不要随便进行，会更新系统内核，可能导致开不了机）

```shell
#yum update　　//升级所有包，同时升级软件和系统内核(#yum upgrade   //升级所有包，不升级软件和系统内核）
```

##### 2、安装依赖包

```sh
#yum install -y yum-utils device-mapper-persistent-data lvm2
```

##### 3、添加`aliyun docker`软件包源

```sh
#yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

##### 4、添加软件包源到本地缓存

```sh
#yum makecache fast
#rpm --import https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
```

##### 5、安装`docker`

```shell
#yum -y install docker-ce
```

##### 6、设置开机启动`docker`

```shell
#systemctl enable docker
```

##### 7、重启`docker`

```shell
#systemctl restart docker
```

=============================================================

##### 添加国内源：修改或新增 `/etc/docker/daemon.json`

```shell
#vim /etc/docker/daemon.json
{
	"registry-mirrors": ["https://pee6w651.mirror.aliyuncs.com"]
}
```

##### 附国内镜像：

docker 官方中国区：https://registry.docker-cn.com

网易：http://hub-mirror.c.163.com

中国科技大学：https://docker.mirrors.ustc.edu.cn

阿里云：https://pee6w651.mirror.aliyuncs.com

```shell
#systemctl restart docker.service
```

## 安装docker-compose

`Docker Compose`项目是`Docker`官方的开源项目，负责实现对`Docker`容器集群的快速编排。`Docker Compose`中的两个重要概念：

- 服务 (service)：一个应用容器，实际上可以运行多个相同镜像的实例

- 项目 (project)：由一组关联的应用容器组成的一个完整业务单元

一个项目可以由多个服务关联（容器）而成，并使用`docker-compose.yml`进行管理。

#### 方法一：

##### 1、下载软件包

```shell
#curl -L https://github.com/docker/compose/releases/download/1.25.5/docker-compose-`uname -s `-`uname -m` > /usr/local/bin/docker-compose
```

#####  2、添加可执行权限

```shell
#chmod +x /usr/local/bin/docker-compose
```

#####  3、检查安装

```shell
#docker-compose -v
```

#### 方法二： 

##### 1、检查`linux`有没有安装`python-pip`包

```shell
#yum install python-pip -y
```

##### 2、没有`python-pip`包就执行命令 

```shell
#yum -y install epel-release
```

##### 3、执行成功之后，再次执行

```shell
#yum install python-pip
```

##### 4、对安装好的`pip`进行升级 

```shell
#pip install --upgrade pip
```

##### 5、安装`docker-compose`

```shell
#pip install docker-compose
```

##### 6、检查安装

```shell
#docker-compose -version
```


