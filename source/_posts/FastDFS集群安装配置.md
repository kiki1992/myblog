---
title: 基于Ngnix的FastDFS集群安装配置
date: 2018-01-03 17:23:32
summary: 本篇简单介绍了开源分布式文件系统FastDFS，并整理了基于Ngnix的集群安装配置上手教程。
tags: [分布式文件系统,FastDFS,Ngnix]
---

### FastDFS简介
> * FastDFS是一个高性能开源分布式文件系统。其主要功能包括：文件存储、同步、访问（上传下载）、解决了大容量存储及负载均衡问题。FastDFS可以满足提供基于文件的服务网站，比如图片/视频分享网站。
>
> * FastDFS中包含两个角色：tracker和storage。
>
>   ***tracker***
>
>   负责调度文件访问及负载均衡。
>
>   ***storage***
>
>   负责存储文件并提供以下文件管理功能：存储、同步、访问接口的提供。同时storage还管理着文件的meta-data信息（以键值对形式体现的文件属性），比如width=1024。
>
>   tracker和storage都可以有一台或多台服务器构成。任何时间从tracker或storage集群中新增或移除服务器都可以避免对线上服务产生影响。其中tracker集群中的所有服务器都是对等的。
>
> * storage服务器通过组织文件卷/组来实现大容量存储。storage系统中可以拥有一个或多个文件卷，各个文件卷的文件之间都是相互独立的。整个storage系统的容量大小等于所有文件卷的总容量。而一个文件卷又可以由一到多台服务器构成，在这些服务器中的文件都是相同的。一个文件卷中的各个服务器会相互备份，并且是负载均衡的。当往某个文件卷中新增一个服务器时，当前文件卷中的文件会被自动复制至新增的服务器，当复制工作结束后，系统将会自动将其切换至线上环境来提供文件存储服务。
>
> * 当storage整体容量不足时，可以增加一个或多个文件卷来实现扩容。
>
> * 一个文件由文件卷名和文件名作为标示。
>
>   ​

以上内容译自(https://github.com/happyfish100/fastdfs)



### FastDFS集群安装配置

#### 环境及软件准备
***安装环境***

这里的软件安装环境采用CentOS6.9，并使用6台服务器来模拟较完整的负载均衡。其中tracker服务器2台，storage服务器四台，storage服务器分为两个文件卷，每个文件卷包含两台服务器。

***软件准备***
libfastcommon -- FastDFS依赖库
[源码地址](https://github.com/happyfish100/libfastcommon)
fastdfs -- FastDFS
[源码地址](https://github.com/happyfish100/fastdfs)
nginx-1.12.2.tar.gz
[前往下载](http://nginx.org/download/nginx-1.12.2.tar.gz)
fastdfs-nginx-module -- FastDFS Nginx模块
[源码地址](https://github.com/happyfish100/fastdfs-nginx-module)

#### 集群安装配置
***基本软件安装***
安装FastDFS之前，需要确保gcc，make等依赖库和工具已经完成安装。

gcc和make等常用工具可以通过以下命令安装：

```shell
yum -y install gcc
yum -y groupinstall Development Tools
```

***libfastcommon安装***

FastDFS依赖于libfastcommon，以下是简单的安装步骤：

