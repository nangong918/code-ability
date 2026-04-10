**Mac项目部署**
====


## 部署流程


### 打包

#### 配置环境
项目中`ruoyi-admin`模块中的yml进行配置环境。

```yml
# Spring配置
spring:
  profiles:
    active: docker-prod
```
测试服将其配置为`docker-test`
正式服务器将其配置为`docker-prod`


#### 打Jar包
* maven sync
![img.png](img.png)

* package
![img_1.png](img_1.png)

* Jar包
![img_2.png](img_2.png)
可以解压检查内部的yml文件中配置的环境是否正确。


### 部署

下载远程连接工具：[MobaXterm](https://mobaxterm.mobatek.net/download.html)

#### 连接服务器

* 测试服
```shell
ssh root@120.78.155.181
password: Ggcjdss168
```


* 正式服
```shell
ssh syncdpt@120.79.182.150
password: Djhjx111
```

DNS：[正式服](mmsys.lednets.com)

#### 重新部署

```shell
cd /home/ruoyiadmin/
./ruoyiadmin.sh
```

#### 上传Jar包

先进入home路径下
```shell
cd /home
```
所有的项目都在home下，ls找到本项目并cd进去
```shell
root@iZwz98kpiilaoou4knjliqZ:/home# ls
ruoyiadmin  ubuntu
root@iZwz98kpiilaoou4knjliqZ:/home# cd ruoyiadmin/
root@iZwz98kpiilaoou4knjliqZ:/home/ruoyiadmin# ls
container_logs.txt  dockerfile  ruoyi-admin.jar  ruoyi-admin.jar.bk  ruoyiadmin.sh
```

删除旧的Jar包，并上传新的的Jar包。
![img_3.png](img_3.png)


#### 安装docker镜像（有则跳过）

检查是否安装了docker
```shell
which docker
docker version
```

如果没有安装则安装docker

```shell
# 安装docker依赖环境
yum install -y yum-utils device-mapper-persistent-data lvm2

# 配置国内docker-ce的yum源（这里采用的是阿里云）
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装docker
yum -y install docker-ce doker-ce-cli containerd.io

# 启动docker并开启自动启动
systemctl start docker && systemctl enable docker

# 查看网络转发功能是否开启，1表示开启
cat /proc/sys/net/ipv4/ip_forward

# 查看docker的版本
docker version

# 查看docker基本信息
docker info
```


#### 安装数据库（如安装则跳过）

```shell
which mysql
mysql -v
```

如果没有数据库则安装数据库
```shell

# 拉取数据库镜像文件
docker pull mysql:5.7.41

# 运行数据库容器
docker run \
--name mysql_common \
-p 4406:3306 \
--restart unless-stopped \
-v /home/mysql_common_v/log:/var/log/mysql \
-v /home/mysql_common_v/data:/var/lib/mysql \
-v /home/mysql_common_v/conf:/etc/mysql/conf.d \
-e TZ=Asia/Shanghai \
-e MYSQL_ROOT_PASSWORD=clt_123456 \
-d mysql:5.7.41

# 查看运行的容器（包含已经停止运行的）
sudo docker ps -a
# 进入数据库容器
sudo docker exec -it mysql_common /bin/bash
# 查看mysql的版本
sudo mysql -V
# 进入数据库
sudo mysql -u root -p

# 输入密码：******
# 开启数据库远程登录
sudo use mysql;

select host,user from user;

ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'clt_123456';

flush privileges;
```

```shell
# 检查宿主机是否有MySQL进程
sudo netstat -tlnp | grep :3306

# 检查是否有MySQL容器在运行
sudo docker ps | grep mysql
```

数据库异常日志
```shell
docker stop mysql8
docker start mysql8
docker exec -it mysql8 bash
```

检查服务器mysql：
```shell
timeout 3 bash -c "cat < /dev/tcp/172.18.213.93/4406" && echo OK || echo FAIL
```

#### 创建dockerfile

为了将 Spring Boot 项目打包到 Docker 容器中，我们需要编写一个 Dockerfile。Dockerfile 是一个简单的脚本，它包含了构建 Docker 镜像所需的所有命令


```shell
# 指定基础镜像，来构建此镜像，可以理解为运行的需要基础环境
FROM openjdk:8
# 维护者信息
MAINTAINER wcs
# 定义匿名卷
VOLUME /tmp
# 创建挂载目录
RUN mkdir -p /ruoyi-admin_workdir/sysRes
#WORKDIR指令用于指定容器的一个目录， 容器启动时执行的命令会在该目录下执行。
WORKDIR /ruoyi-admin_workdir
##将当前jar包复制到容器对应目录下并修改名称
ADD ruoyi-admin.jar /ruoyi-admin_workdir/app.jar
# 允许指定的端口
EXPOSE 8186
# 入口
ENTRYPOINT ["java","-jar","/ruoyi-admin_workdir/app.jar", "-Duser.timezone=GMT+8","--spring.profiles.active=docker-test"]
```


#### 构建docker镜像


```shell
cd ../
cd /home/ruoyiadmin
# 在项目目录下执行：docker build -t springboot-app .
docker build -t springboot-app .

# 查看本地镜像列表，能看到springboot-app即为确认成功
docker images | grep springboot-app
```


#### 运行docker容器


```shell
# 首先查看docker端口占用
sudo docker ps -a --format "table {{.Names}}\t{{.Ports}}"

# 先删除旧容器（数据在镜像和卷里，不会丢）
sudo docker rm -f ruoyiadmin
sudo docker rm ruoyiadmin:v1.0

docker build -t ruoyiadmin:v1.0 ./

# -p 8186:8186：将容器的 8186 端口映射到宿主机的 8186 端口
# --name ruoyiadmin:v1.0：给容器指定一个名称
docker run -p 8186:8186 -v /data/docker/ruoyiadmin/sysRes:/ruoyi-admin_workdir/sysRes --name ruoyiadmin -d --restart unless-stopped  ruoyiadmin:v1.0

# 查看日志
docker logs --tail 1000 ruoyiadmin > container_logs.txt
docker logs ruoyiadmin:v1.0
```

最后就可以看到容器启动成功。
测试服直接访问ip，正式服访问其域名。


### 问题排查

```shell
docker logs --tail 100 ruoyiadmin
```



### 数据库查看

首先查询数据库：
```shell
docker ps | grep -E "mysql|mariadb|postgres"

# 可以看到

5c07f724ef12   mysql:8.0           "docker-entrypoint.s…"   3 months ago     Up 5 days       33060/tcp, 0.0.0.0:3307->3306/tcp, :::3307->3306/tcp   mysql-ruoyi
6b10740e1cb0   mysql:8.0.28        "docker-entrypoint.s…"   13 months ago    Up 5 days       33060/tcp, 0.0.0.0:3706->3306/tcp, :::3706->3306/tcp   mysql8-v2
0a7e22cf5d14   mysql:5.7.41        "docker-entrypoint.s…"   2 years ago      Up 5 days       0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   mysql_common

```

进入数据库
```shell
docker exec -it mysql-ruoyi /bin/bash

# 登录容器内的 MySQL
mysql -u root -p

# 输入密码：******
```

