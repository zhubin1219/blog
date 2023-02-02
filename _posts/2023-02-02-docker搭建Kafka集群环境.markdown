---
title:  "docker搭建Kafka集群环境"
date:   2023-02-01 00:00:00
categories: [docker,kafka]
tags: [环境搭建]
---

# 本机 window11 docker搭建Kafka集群环境
## 前因后果  
之前项目用到kafka集成，但是是在别人环境搭建好的基础上开发，对基础环境的建立以及前后了解不足，故闲来无事搭建环境更加深入了解下，本来打算在云服务器上安装环境测试， 不过这样只能搭建单机kafka环境，用虚拟机多开的话自己的小破服务器估计也吃不消，故想到了docker,之前也是耳闻没有静下心了解，借此机会，一并学习下  

## 基础知识  
网络上相关资料和教程很多，看了很多，对于本次目标需要的知识重点说明，其他的提一嘴，后面用到了学习后在来填坑  

**首先看一个简单的docker架构图，后续我们也会根据这个思路完成我们的环境搭建**

|名词|说明
|:---:|:---:
|Client|就是客户机,会输入命令给到docker环境
|docker_host|就是docker安装后的环境，可以识别docker 的相关命令，docker daemon(守护进程)会识别 client输入的命名，完成docker操作，建立image，获取仓库image,用image运行容器等
|registry|仓库，官方和个人可以在这里推送自己的 images,这样就可以直接那来使用，不用自己建立image，后面搭建用到的kafka image也都是直接获registry仓库获取的 


![docker架构](../../images/blog/20230201/微信图片_20230202135710.png)

## docker环境安装  

> 此处根据微软官方文档[window 安装WSL 2](https://learn.microsoft.com/zh-cn/windows/wsl/install)  

### window11 环境安装  

**1. 身为.net开发平台的开发者，常年使用window平台，但是实际上docker是运行在linux环境下的，所以我们需要让window有linux环境，在此基础上才可以使用docker**  

**2. window平台提供了二种方案，一种是基于WSL 2（通过适用于 Linux 的 Windows 子系统 (WSL)，开发人员可以安装 Linux 发行版）的实现，一种是Hyper-V的实现**  

**3. 本教程使用WSL2实现，主要Hyper-V又一个坑，如果window开启此功能，那么低版本安卓模拟器和虚拟机将无法正常运行，存在冲突**  

**4. 涉及命令如下，管理员权限Windows PowerShell 执行**

启用适用于linux的window系统（其实我理解的是适用window系统的linux系统...不过官网这么说来着）
`dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart`  

启用虚拟机功能，安装 WSL 2 之前，必须启用“虚拟机平台”可选功能。 计算机需要虚拟化功能才能使用此功能。
`dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart`  

**重新启动计算机，以完成 WSL 安装并更新到 WSL 2**

下载linux内核更新包并安装
[适用于 x64 计算机的 WSL2 Linux 内核更新包](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)


将WSL 2设置为默认版本
`wsl --set-default-version 2`

至此docker安装的 window11 所需要的依赖环境安装完毕  

### Docker Desktop for Windows 安装  
[Docker Desktop for Windows 下载文件以及相关教程](https://docs.docker.com/desktop/install/windows-install/)  

安装没啥特殊的，默认值一直安装结束就好，出现如下画面就证明docker环境安装成功
![docker安装成功](../../images/blog/20230201/微信截图_20230202145028.png)  

## docker 的一些命令参数了解  

本教程主要了解docker Run 一些参数即可  

|参数|说明
|:---:|:---:
|--name|给启动的容器一个名字
|-i|即使未连接STDIN（标准输入）也保持打开状态，分配一个交互终端
|-t|示容器启动后会进入其命令行，与it一起使用,分配一个伪tty设备,可以支持终端登录
|-d|后台运行并且打印容器ID
|-p|端口映射，前面为宿主机的端口，后面为容器服务进程端口，访问宿主机的80,最终会转发给容器的80端口
|-e|给容器声明环境变量，在容器内部可以通过export查看  

虽然我们的目标只是搭建kafka集群环境，只需要用到部分命令，但是多了解些也不会亏不是
[docker命令详解](https://docs.docker.com/engine/reference/commandline/docker/)

## docker 搭建 kafka 环境

本教程准备搭建一台Zookeeper以及三个Kafka broker组成的Kafka Cluster,顺序为先建立Kafka依赖Zookeeper(文件在各个集群机器备份)，然后安装kafka环境

**1. 安装Zookeeper**
可以使用 `docker search zookeeper` 查找Zookeeper也可以直接使用 docker Desktop Search 直接查找相关image，网上全是linux相关命令，这里我就直接贴docker Desktop Search结果了
![zookeeperImage](../../images/blog/20230201/微信截图_20230202145028.png)

**理论上镜像我们都第一选择选择官方镜像，不过考虑到后续kafka的镜像，这里使用了同一作者发布的Zookeeper image**
`docker pull wurstmeister/zookeeper` 也可以docker Desktop 直接pull，效果一样

**启动Zookeeper镜像**  
`docker run -d --name zookeeper -p 2181:2181 -t wurstmeister/zookeeper`

***此处有个概念，image是镜像，只用拉取一次，根据镜像可以产生N个容器，每个容器有自己的Volumes(持久化存储资料)**
所以我们根据wurstmeister/zookeeper image 运行了一个容器，这个容器会记录每次操作的数据，下次启动该容器资料上次操作的资料还在，很赞对吧~

**拉取kafka镜像**  
`docker pull wurstmeister/kafka` 也可以docker Desktop 直接pull，效果一样  

**根据 wurstmeister/kafka image 启动三个容器 模拟三个服务器broker**
**BROKER 0**
`docker run -d --name kafka0 -p 9092:9092 -e KAFKA_BROKER_ID=0 -e KAFKA_ZOOKEEPER_CONNECT=192.168.0.101:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.0.101:9092 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 -t wurstmeister/kafka
`  
**BROKER 1**
`docker run -d --name kafka1 -p 9093:9093 -e KAFKA_BROKER_ID=1 -e KAFKA_ZOOKEEPER_CONNECT=192.168.0.101:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.0.101:9093 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9093 -t wurstmeister/kafka
`  
**BROKER 2**  
`docker run -d --name kafka2 -p 9093:9094 -e KAFKA_BROKER_ID=2 -e KAFKA_ZOOKEEPER_CONNECT=192.168.0.101:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.0.101:9094 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9094 -t wurstmeister/kafka
`  

## 程序模拟生产者，消费者，测试集群环境

**生产者分别给三个broker 发送消息**  
![生产者消息](../../images/blog/20230201/微信截图_20230202155029.png)

**消费者监听其中一个broker 9092端口，实际获取到三个生产者发送的消息，证明集群消息已经同步，接收成功，模拟成功~**  
![消费者消息](../../images/blog/20230201/微信图片_20230202155236.png)  

**模拟结果**  
![模拟结果](../../images/blog/20230201/微信截图_20230202153618.png)  

还记得Volumes 不，此处结果为容器启动模拟关闭后，再次模拟的情况，发送消息的ID没有重新累计，而是从之前序号流水往下，就是Volumes 的作用了（目前理解）

**至此，我们的目标已经达成,当然docker 和 kafka的知识远不止于此，这里只是简单了解，为后续更好的去应用打下基础**
