---
layout: post
title: Android应用框架之Activity
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
上一篇博客介绍了`Application`和`ActivityThread`，今天来讲一下Android中最为重要的一个组件,`Activity`。

### 1.基本结构
一个应用程序通常由多个`Activity`组成，那么在应用程序中肯定需要一个容器来盛放这些`Activity`，必要时通过该容器找到对应的`Activity`，并进行相关操作。上一篇文章已经讲过一个应用程序对应一个`ActivityThread`，所以自然而然地该容器是`ActivityThread`在负责维护，这个容器叫做`mActivities`，是一个数组，里面的每一项叫做`ActivityRecord`，一个`ActivityRecord`对应一个`Activity`。以上仅仅是应用级别的管理容器，但是很多场景下，系统需要找到某一个特定的`Activity`，并下发相关数据比如事件分发。所以还必须在系统层面再维护一个容器，这个容器存放在`Activity Manager Service`,对应的容器叫做`mHistory`,对应的每一项叫做`HistroyRecord`。
每个`Activity`必须依靠在进程中，每个进程对应一个AMS中的`ProcessRecord`，通过这个`ProcessRecord`可以找到对应的应用的所有`Activity`，同时还提供了与`Activity`联系的接口`IActivityThread`。所以整个`Activity`的管理框架如下图所示：
![这里写图片描述](http://img.blog.csdn.net/20161027225650036)

### 2.`Activity`启动过程
在Launch Activity时，AMS将对应的HistoryRecord作为token传递到客服端和客服端的Activity建立联系。在AMS中Activity状态变化时，将通过该联系找到客服端的Activity，从而将消息或者动作传递应用程序面对的接口：xxxActivity。整个`Activity`的启动过程大致可以分为以下几个步骤：
- 发起`startActivity（intent）`请求
- AMS接收到请求后，创建一个`HistroyRecord`对象，并将该对象放到`mHistory`数组中
- 调用`app.thread.scheduleLaunchActivity()`
- AMS创建`ActivityRecord`对象，将创建的`Activity`放入到`ActivityRecord`，再将其放入到`mActivities`
- 发起`Activity`的`onCreate（）`方法

对应的步骤如下图所示：
![这里写图片描述](http://img.blog.csdn.net/20161027225734467)