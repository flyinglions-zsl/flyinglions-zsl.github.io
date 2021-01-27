---
title: rabbitmq01
date: 2021-01-27 22:39:04
categories:
  - rabbitmq
tags:
  - rabbitmq
---

# 常用指令

docker images

docker pull

docker run 

​	-d 后台运行

​	-p 容器对主机开放端口 如：docker run -d -p 8080:80 REPOSITORY 容器的80端口对主机开放,且主机访问端口需为8080

​	-P 容器对主机开放所有端口 如：docker run -d -P REPOSITORY 容器80端口对主机开放，但主机会随机生成一个端口供访问

docker ps 查看正在运行的容器

docker stop containerid 停止某个在运行的容器

docker exec -it container bash 查看容器的目录结构，类似一个linux系统了

​	bash里面无法使用 ps -ef ：apt-get update && apt-get install procps 安装

netstat -an|grep 端口

docker 网络(规则)：

​	网络类型：

​		bridge 和主机通过网桥连接，容器有独立的端口和ip，访问主机时通过网桥，再端口映射到(容器)机器上，再原路返回结果

​		host  和主机的网卡直接相连 ip和主机一致

​		none



# 制作镜像

1.Dockerfile

vi Dockerfile

编译： docker build -t demo:latest .

2.docker build

docker ps -a

docker rm ps的id

docker rmi containerid

3.jpress.io