#Doker的基本使用及遇到的坑

[TOC]

## 一、Docker的安装

### MacOS Docker 安装

1. **方法一**：可以使用 Homebrew 来安装 Docker。输入命令：brew cask install docker即可；
2. **方法二**：网上下载安装包之后安装。

### Docker配置

在设置-Docker Engine下：

1. 在insecure-registries中添加私有镜像仓库地址；

2. 在registry-mirrors中最好换上国内镜像，加快镜像拉取速度。

   比较常用的是网易的镜像中心和daocloud镜像市场。 
   网易镜像中心：https://c.163.com/hub#/m/home/ 
   daocloud镜像市场：https://hub.daocloud.io/

具体配置参考下图：

![image-20201217143200798](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20201217143200798.png)



## 二、Java打包镜像

###基础命令打包：

1. 创建Dockerfile，具体配置如下：

```
FROM 47.114.174.69:5000/luqi-openjdk:8-jdk-alpine

LABEL evaluation Sherry
#ENV 变量

ENV TZ='Asia/Shanghai'
ENV DATA_API_ADDRESS=""

#COPY
COPY   target/epidemic-analysis-core.jar /root

#EXPOSE 服务端口

EXPOSE 9002

#ENTRYPOINT 启动命令
ENTRYPOINT java -jar  /root/epidemic-analysis-core.jar
```

2. 先把程序打成jar包；
3. Terminal输入打包命令：docker build -t XXXX/XXXX .（**别忘记"."**）

###Idea配置docker

1. Preferences（设置）-Plugins，下载Docker插件，点击Install

   ![image-20201217145012177](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20201217145012177.png)

2. 增加Docker配置，选择Docker-Dockerfile：

   ![image-20201217150040028](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20201217150040028.png)

   配置详情如下：

   ![image-20201217150333162](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20201217150333162.png)

3. 运行即可打包完成

###docker基础命令

上传：docker push  XXXX/XXXX

进入容器：docker exec 【id】sh

查看容器文件：docker exec 【id】ls 【路径】

查看镜像：docker images

切换私有镜像仓库：docker login 【私有镜像路径】

查看所有正在运行容器：docker ps 
停止某个容器：docker stop 【容器的ID】



## Docker打包遇到的坑

### 1. 没有中文字体导致报NullPointerException

在部署过程中有两个项目遇到莫名其妙的问题，1.根据名字首字生成头像调用内部方法报错，2.创建Excel，调用内部方法报错。经过排查，发现是字体的原因，打包时候的基础镜像不包含字体，解决方法如下：

1. 打包一个含有中文字体的基础镜像
2. 打包的时候依赖这个含有字体的基础镜像

具体参考文章：https://juejin.cn/post/6844903952824156167



### 2. 本地存储的文件，重新部署之后文件没了

原因是docker上的文件在容器内部，docker重新部署之后，文件就没有了。

解决方法，docker容器映射目录：

docker run -p 8079:80 --name nginx-test 
--privileged=true 
-v /testdocker/default.conf:/etc/nginx/conf.d/default.conf 
-v /testdocker/html:/usr/share/nginx/html -d nginx:1.14

-p: 指定端口映射，格式为：主机(宿主)端口:容器端口
--privileged=true 关闭安全权限，否则你容器操作文件夹没有权限
-v 挂载目录为：主机目录:容器目录，在创建前容器是没有指定目录时，docker 容器会自己创建



## 总结

本文简单总结了下本次上云用到Docker的一些基本操作，记录了两个问题。后续如果有深入研究，再继续补充。







