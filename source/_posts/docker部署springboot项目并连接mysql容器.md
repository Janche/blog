---
title: docker部署springboot项目并连接mysql容器
comments: true
date: 2019-04-26 15:03:13
tags:
    - docker
    - springboot
---

## 1. 先拉取mysql镜像(因为比较慢)
`docker pull mysql:5.7`
## 2. 构建要部署的项目镜像
### 2.1 创建一个目录
<!-- more -->
**root@lr-pc:/usr/local/docker# `mkdir docker-web`**
**1. 将需要部署的jar包拷贝到此目录下: `mv demo-pmsd.jar /usr/local/docker/docker-web`

2. 新建一个Dockerfile文件: `vim Dockerfile`**
Dockerfile文件内容：

```
# 基础镜像使用Java
FROM java:8
# 作者
MAINTAINER lr
# VOLUME 指定了临时文件目录为/tmp。
# 其效果是在主机 /var/lib/docker 目录下创建了一个临时文件，并链接到容器的/tmp
VOLUME /tmp
# 将jar包添加到容器中并更名为app.jar
ADD demo-pmsd.jar app.jar
# 运行jar包
RUN bash -c 'touch /app.jar'
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
# 指定容器需要映射到主机的端口
EXPOSE 8080
```
### 2.2 执行镜像构建命令
**`docker image build -t demo-pmsd .` // 镜像名随意，注意最后有一个点**
>注意：需要在Dockerfile目录下执行， `.`表示在当前目录，也可指定目录。
>可通过 `docker images`查看刚刚构建的镜像
### 3. 开启mysql服务
#### 3.1 首先你需要在你的ubuntu系统上装好mysql，应该都装了把，没有的伙伴可以看我这篇教程：[ubuntu18.04安装mysql5.7](https://blog.csdn.net/qq_34997906/article/details/83046680)
#### 3.2 将你要部署项目的数据导入ubuntu的mysql(就是建数据库和导数据)
#### 3.3 (重点)用刚刚拉取的mysql镜像启动容器
```
docker run -p 3306:3306 --name db -v /etc/mysql/conf:/etc/mysql/conf.d -v /usr/local/docker/mysql/logs:/logs -v /var/lib/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -d -it mysql:5.7
```
**命令解释：**
```
# 容器终止运行后自动删除容器文件
# --rm
# 主机端口映射到容器端口
#-p 3306:3306 
# 给容器起别名（非常重要，项目中的数据库地址需要和别名一致）
#--name db 
# 把主机的配置文件映射到容器的配置文件
#-v /etc/mysql/conf:/etc/mysql/conf.d 
# 把主机的日志映射到容器的日志
#-v /usr/local/docker/mysql/logs:/logs 
# 把主机的数据映射到容器（每次重启容器不用担心数据被清空了）
#-v /var/lib/mysql:/var/lib/mysql 
# 数据库密码
#-e MYSQL_ROOT_PASSWORD=root 
# 后台启动
#-d 
# 容器的 Shell 映射到当前的 Shell，然后你在本机窗口输入的命令，就会传入容器。
#-it 
# 来自哪一个镜像
#mysql:5.7
```
>**建议将此命令，写入一个可执行的文件，方便以后使用**
### 4. 用刚生成的项目镜像启动容器
`docker run --rm -d -p 8081:8080 --name demo-pmsd --link db:db demo-pmsd`
**命令解释：**
```
# 容器终止运行后自动删除容器文件
#--rm
# 后台启动
#-d 
# 主机端口映射到容器端口
#-p 8081:8080 
# 为容器起别名
#--name demo-pmsd
# 连接提供mysql服务的容器，冒号后面是别名，别名应该和代码中的数据库地址一致(这点真的很重要)
#--link db:db
# 由哪个镜像生成的
#demo-pmsd
```
>**同样建议将此命令，写入一个可执行的文件，方便以后使用**

### 5.恩，没有然后了，你可以在浏览器端访问你的项目了`localhost:8081`。

>参考：[docker常用命令](https://blog.csdn.net/qq_34997906/article/details/83002327)