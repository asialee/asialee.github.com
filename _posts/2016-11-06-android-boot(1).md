---
layout: post
title: Android启动过程详解（1）——总体启动框架
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
在接下来的几篇博客中我会主要给大家讲一下Android系统的启动过程，之前也断断续续讲过`PackageMangerService`和Home程序的启动过程，但是没有系统的讲过，接下来将系统性地介绍整个系统的启动过程。包括主要的四大步骤：

> 1. init进程服务
> 2. Native服务启动
> 3. SystemServer，Android服务启动
> 4. Home应用程序启动

总体的启动流程如下图所示：
![这里写图片描述](http://img.blog.csdn.net/20161106191355653)
接下来的《Android启动过程详解》系列将以本图为框架进行介绍，重点包括四大步骤，以及各个步骤中的重点组件：`Zygyte`,`ServiceManager`,`Dalvik VM`, `SystemServer`,`ActivityManagerService`,`PackageManagerService`等等。

本文首先对整体的启动过程做一个简单的梳理。

### 1.initial进程(system/core/init)
init是一个由内核启动的用户级进程，在Linux内核启动完成后就会启动init进程，再由init进程来完成接下来加载过程。可以说init进程的启动才标志着Android空间的启动。Init进程一起来就根据init.rc和init.xxx.rc脚本文件建立了几个基本的服务：
> (1) ZygoteService
> (2) ServiceManager
> ......

这里的Service并不是Android应用层的Service概念，而是为整个系统服务的基础服务，其实现也是在Native层完成的。Init.rc是Android规定的初始化脚本，其包括了四种类型：
> 1. Service
> 2. Commands
> 3. Options
> 4. Actions

而init程序正是通过读取init.rc的文件来读取相关配置：

- 读取init.rc的相关服务，添加到service_list中
- 通过service_start，建立service进程

### 2.ZygoteService&ServiceManager

在系统启动的第二步中，init会启动一系列的Native Service，这其中最重要的就是ZygoteService和ServiceManager两个组件，通过[Android应用框架之应用启动过程](http://blog.csdn.net/asialiyazhou/article/details/53049894)一文可以知道，ZygoteService最重要的作用就是通过socket监听来自AMS的应用创建指令，然后fork出新的进程作为新的应用进程。而ServiceManager则是管理各个Android系统级Service。

### 3.SystemServer
在创建好ZygoteService之后，它就会fork出一个进的进程来运行SystemServer，而SystemServer最为重要的一个工作就是启动各个系统级的Android应用服务，包括PackageManagerService， ActivityManagerService，WindowManagerService等等。SystemServer会开启一个ServerThread，然后在这个线程中启动上述各个Service，并通过addService添加对应的服务。

### 4.Home应用程序
`PackageManagerService`在启动后会读取所有应用程序的注册信息，而`ActivityManagerService`会在启动后启动Home应用程序，具体的过程参见：[Android应用程序框架之Home程序（Launcher）](http://blog.csdn.net/asialiyazhou/article/details/53048365)
