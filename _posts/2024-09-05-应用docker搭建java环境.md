---
layout: post
title: docker
categories: [云原生]
description: 应用docker搭建java环境
keywords: docker, 腾讯云, centos7
---

# 一、Docker 生态架构及部署

## 1.1 生态架构

![image-20240905094604485](/images/posts/image-20240905094604485.png)



### 1.1.1 Docker Host

用于安装Docker Daemon的主机，即为Docker Host，并且该主机中可基于容器镜像运行容器。

### 1.1.2 Docker Daemon

用于管理Docker Host中运行的容器、容器镜像、容器网络等，管理由Containerd.io提供的容器。

### 1.1.3 Registry

容器镜像仓库。

### 1.1.4 Docker Client

Docker Daemon客户端工具，用于同Docker Daemon进行通信，执行用户指令，可部署在Docker Host上，也可以部署在其它主机，能够连接到Docker Daemon即可操作。

### 1.1.5 Image

Docker镜像, 把应用运行环境及计算资源打包方式生成可再用于启动容器的不可变的基础设施的模板文件，主要用于基于其启动一个容器。

### 1.1.6 Container

由容器镜像生成，用于应用程序运行的环境，包含容器镜像中所有文件及用户后添加的文件，属于基于容器镜像生成的可读写层，这也是应用程序活跃的空间。

### 1.1.7 Docker Dashboard

管理Docker的简单页面。



# 二、Docker 部署

## 2.1 使用yum部署(推荐)

### 2.1.1 获取阿里云yum源镜像

```powershell
1. 在docker host上使用 wget下载到/etc/yum.repos.d目录中即可。
# wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

2. 查看当前主机的yum源
# yum repolist

```

### 2.1.2 安装Docker-ce

```powershell
直接安装docker-ce，此为docker daemon，所有依赖将被yum自动安装，含docker client等。
# yum -y install docker-ce
```

### 2.1.3 配置Docker Daemon启动文件

```
# vim /usr/lib/systemd/system/docker.service

先删除图中所示代码
再下面一行新增加
ExecStartPost=/sbin/iptables -P FORWARD ACCEPT
```

![image-20240905104455719](/images/posts/image-20240905104455719.png)

### 2.1.4 启动Docker服务并查看已安装版本

```powershell
重启加载daemon文件
# systemctl daemon-reload

启动docker daemon
# systemctl start docker

设置开机自启动
# systemctl enable docker

查看docker版本
# docker version
```

### 2.1.5 容器镜像加速器

```powershell
添加daemon.json配置文件
# vim /etc/docker/daemon.json
# cat /etc/docker/daemon.json
{
        "registry-mirrors": ["https://s27w6kze.mirror.aliyuncs.com"]
}

重启docker
# systemctl daemon-reload
# systemctl restart docker

```



# 三、Docker命令

## 3.1 Docker命令获取帮助方法

```powershell
# docker -h
```



## 3.2 Docker官网提供的命令说明

```powershell
https://docs.docker.com/reference/
```



## 3.3 Docker常用命令

### 3.3.1 容器相关

```powershell
1. docker run
# docker run -i -t --name c1 centos:latest bash

命令解释
docker run 运行一个命令在容器中，命令是主体，没有命令容器就会消亡
-i 交互式
-t 提供终端
--name c1 把将运行的容器命名为c1
centos:latest 使用centos最新版本容器镜像
bash 在容器中执行的命令

查看网络信息
# ip a s

查看进程
# ps aux

查看用户
# cat /etc/passwd

查看目录
# pwd

退出命令执行，观察容器运行情况
# exit


2. docker ps
# docker ps
命令解释
docker ps 查看正在运行的容器

# docker ps --all


3. docker inspect
# docker inspect 容器名称
查看容器详细信息


4. docker  exec 
# docker exec -it c2 ls /root
命令解释
docker exec 在容器外实现与容器交互执行某命令
-it 交互式
c2 正在运行的容器名称
ls /root 在正在运行的容器中运行相关的命令


5. docker attach
# docker attach c2
命令解释
docker attach 类似于ssh命令，可以进入到容器中
c2 正在运行的容器名称
退出容器时，如不需要容器再运行，可直接使用exit退出；如需要容器继续运行，可使用ctrl+p+q

6. docker stop
# docker stop 容器id

7. docker start
# docker start 容器id

8. docker top
# docker top 容器名
命令解释
docker top 查看container内进程信息，指在docker host上查看，与docker exec -it c2 ps -ef不同。


9. docker rm
容器停止后, 删除容器
# docker stop 容器名称/id
```

### 3.3.2 镜像相关

```powershell
1. 查看
# docker images
# docker image list
查看docker容器镜像本地存储位置
# ls /var/lib/docker

2. 搜索docker hub镜像
# docker search 镜像名/id

3. 镜像下载
# docker pull 镜像名/id

4. 镜像删除
# docker rmi 镜像名/id

5. 将容器制作成镜像
# docker commit 容器名/id
6. 导出镜像导出为tar压缩包
# docker save -o centos.tar centos:latest  
7. 把正在运行的容器导出
# docker export -o centos7.tar 355e99982248
8. 将镜像作为本地镜像
# docker import centos7.tar centos7:v1


```



# 四、Docker Harbor(pending)

## 4.1 获取 docker compose二进制文件

```powershell
下载docker-compose二进制文件
# wget https://github.com/docker/compose/releases/download/1.25.0/docker-compose-Linux-x86_64
查看
# ls
移动文件并更名
# mv docker-compose-Linux-x86_64 /usr/bin/docker-compose
为二进制文件添加可执行权限
# chmod +x /usr/bin/docker-compose
安装完成后，查看docker-compse版本
# docker-compose version
```



## 4.2 获取harbor安装文件

```powershell
下载harbor离线安装包
# wget https://mirror.ghproxy.com/https://github.com/goharbor/harbor/releases/download/v2.4.1/harbor-offline-installer-v2.4.1.tgz
```



# 五、企业级应用安装(对外暴露端口, 对外端口采用非默认)

## 5.1 Nginx

```powershell
# docker run -d -p 7001:80 --name nginx -v /opt/nginx:/usr/share/nginx/html:ro nginx
增加Ng访问页面
# echo "nginx is running" > /opt/nginx-server-port/index.html
宿主机直接访问
```



## 5.2 Tomcat

```powershell
# docker run -d -p 7002:8080 --name tomcat -v /opt/tomcat:/usr/local/tomcat/webapps/ROOT tomcat:9.0
添加Tomcat访问页面
# echo "tomcat running" > /opt/tomcat-server/index.html
宿主机直接访问
```



## 5.3 Mysql8(单节点部署)

```powershell
1. 拉取镜像
# docker pull mysql:8.0.20
启动镜像
# docker run -p 7003:3306 --name mysql8 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:8.0.20

2. 配置挂载目录
# mkdir -p /opt/mysql8.0.20/
拷贝镜像下的配置文件
# docker cp  mysql8:/etc/mysql /opt/mysql8.0.20/
删除原有镜像
# docker stop mysql8
# docker rm mysql8
启动mysql挂载配置文件，数据持久化到宿主主机
# vim /opt/mysql8.0.20/mysql/conf.d/my.cnf

[mysqld]
user=mysql
character-set-server=utf8
default_authentication_plugin=mysql_native_password
secure_file_priv=/var/lib/mysql
expire_logs_days=7
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
max_connections=1000
 
[client]
default-character-set=utf8
 
[mysql]

创建shell脚本, 记录启动参数设置
# vim /opt/mysql8.0.20/mysql/docker_insert_mysql8.0.20.sh

#!/bin/sh
docker run \
-p 7003:3306 \
--name mysql8 \
--privileged=true \
--restart unless-stopped \
-v /opt/mysql8.0.20/mysql:/etc/mysql \
-v /opt/mysql8.0.20/logs:/logs \
-v /opt/mysql8.0.20/data:/var/lib/mysql \
-v /etc/localtime:/etc/localtime \
-e MYSQL_ROOT_PASSWORD=123456 \
-d mysql:8.0.20

通过shell脚本启动
sh /opt/mysql8.0.20/mysql/docker_insert_mysql8.0.20.sh


3. 配置链接信息
# docker exec -it mysql8 bash
进入mysql, 配置权限相关问题
#  mysql -u root -p
grant all PRIVILEGES on *.* to root@'%' WITH GRANT OPTION;

use mysql;
update user set host='%' where user='root';
(ERROR 1062 (23000): Duplicate entry '%-root' for key 'user.PRIMARY'提示异常表明已存在用户, 查询下)
SELECT host, user FROM user WHERE user = 'root';

4. 更新密码算法, 便于外部连接
grant all PRIVILEGES on *.* to root@'%' WITH GRANT OPTION;
ALTER user 'root'@'%' IDENTIFIED BY 'xxx' PASSWORD EXPIRE NEVER;
ALTER user 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'xxx';
FLUSH PRIVILEGES;

```



## 5.4 Redis(单节点部署)

```powershell
# mkdir -p /opt/redis/conf
# touch /opt/redis/conf/redis.conf
# docker run -p 7004:6379 --name redis -v /opt/redis/data:/data \
-v /opt/redis/conf:/etc/redis \
-d redis redis-server /etc/redis/redis.conf

更改redis密码
docker exec -it redis bash
cd /usr/local/bin
redis -cli
config get requirepass
config set requirepass xxx
```

