---
layout: post
title: Android应用框架之Application&ActivityThread
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
不同于其他系统，在Android中Application并不是一个重要的概念，甚至开发人员在开发的过程中很少需要直接与Application打交道，其提供的也仅仅是一个上下文环境。至于为什么会这样，还是与设计者的设计思想有关。Android的设计者希望呈现给用户的是一个组件化的操作系统，开发者只需要与`Activity`,`Service`,`BroadcatReceiver`,`ContentProvider`进行交互就行了，并且上述这些组件才是Android的核心概念。
但是Android毕竟是给予Linux的，一个应用程序需要跑起来也必须依托于一个进程之上，所有的组件也必须寄宿在某一个进程之上。所以Application实际上是一个容器或者宿主，用于盛放各个组件。

当第一个`Activity`被创建时，系统会为通过`makeApplication`方法创建一个Application实例，同时也会创建一个进程。默认情况下这个进程的名字和包名相同。当然开发者也可以通过```android:process=name```的方式设定进程名。之后再创建其他组件时，也会被放入到这个进程中。

### ActivityThread
尽管Application名字听起来很唬人，但实际上真正的应用框架在ActivityThread：
> NaiveStart.main()

>  ZygoteInit.main

>  ZygoteInit$MethodAndArgsCall.run

>  Method.Invoke

>  method.invokeNative

>  ActivityThread.main()

>  Looper.loop()

>  ....

以上是应用的启动堆栈，从中可以看到真正的入口是`ActivityThread`。
应用程序以```ActivityThread.main()```为入口启动，同时进入到```Looper.loop()```创建的消息循环中。作为一个人机交互的系统，一个应用最为核心的就是消息循环与事件分发处理，而消息循环就在`ActivityThread`中。除此之外，```ActivityThread```还提供了一个```IActivityThread```接口给```Activity Service Manager(AMS)```作为与AMS通信的接口。

接下来来看看```ActivityThread```是如何创建的。实际上```ActivityThread```与```Activity Manager Service```的关系和```ViewRoot```与```Window Manager Service```的关系很类似。而创建```ActivityThread```的过程就有点类似于```ViewRoot```的```addView```。其过程大致可以分为两个步骤：
- 1 Activity Manager创建进程
- 2 被创建的进程attach到Activity Manager，建立与AM之间的联系

##### ActivityManager创建进程
在AM本地用ProcessRecord来指代一个应用进程。同时AMS还维护两个数组：mProcessNames以及mPidsSelfLocked。两者存储的对象如下图所示：![这里写图片描述](http://img.blog.csdn.net/20161027225151721)
![这里写图片描述](http://img.blog.csdn.net/20161027225209850)
- 1）Activity Manager Service首先通过```startProcessLocked(processName,Appinfo.uid)```创建一个进程对象，类型是ProcessRecord，加入到mProcessNames数组中。在这个数组中应用通过包名和uid标识自己，如果在同一个Package中的Activity,如果都使用默认设置，那么这些Activity都会托管在同一个进程中，这是因为他们在带的ApplicationInfo中的ProcessName都是一样的。
- 2）android.app.ActivityThread进程启动。Android.app.ActivityThread进程建立后，将跳入到ActivityThread的main函数开始运行，进入消息循环。应用进程使用thread.attach()发起AMS的AttachApplicationLocked调用，并传递 ActvitiyThread对象和CallingPid。AttachApplicationLocked将根据CallingPid在mPidsSelfLocked找到对应的ProcessRecord实例app,将ActvitiyThread放置在app.thread中。这样应用进程和AMS建立起来双向连接。AM可以使用AIDL接口，通过app.thread可以访问应用进程的对象。
- ![这里写图片描述](http://img.blog.csdn.net/20161027225314737)

