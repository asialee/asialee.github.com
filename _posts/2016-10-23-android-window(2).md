---
layout: post
title: Android窗口管理（2）——窗口基本架构
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
本文内容参考：[Android 核心分析(13) -----Android GWES之Android窗口管理](http://blog.csdn.net/maxleng/article/details/5557758)
上一篇文章主要讲述了窗口的基本结构，那么在这样的结构下，系统如何管理窗口，如何下发事件，如何获取窗口状态？这篇博客将对这部分的内容进行介绍。
Android在窗口管理上采用了最为经典的C/S模式，Client端是各个Activity中的window，而Service端就是系统持有的窗口管理器Window Manager。

### 总体结构
`Window`是顶级的窗口概念，而`Activity`中的`DecorView`则是窗口中的顶级`View`,创建`Activity`时，`DecorView`会attach到`Activity`的窗口中，同时也被加入到`WindowManager`中，`WindowManager`使用`WindowState`与该`View`相对应。
![这里写图片描述](http://img.blog.csdn.net/20161022224606165)
两者之间通过建立session会话进行通信，而这里的session采用的还是Android中最重要的IPC方式——AIDL。`Activity`在建立窗口后需要将该窗口注册到`WindowManager`中，这个过程涉及到在`Activity`本地创建一个`WindowManager`的代理，`Activity`通过这个代理和远程`WindowManager`进行会话，会话的通道是`IWindowSession`，本质上就是一个AIDL通信过程。![这里写图片描述](http://img.blog.csdn.net/20161022231159693)
会话是双向的，为了将消息发送给对应的`Window`,`WindowManager`通过`IWindow`接口将对应的消息发送给`Window`端对应的处理函数。

### Client端——Activity
首先来看一下`Client`端，通过我的上一篇博客[Android窗口管理（1）——窗口基本架构](http://blog.csdn.net/asialiyazhou/article/details/52892297)我们知道了`Activity`端的窗口结构，并且知道了`Window`,`PhoneWindow`都不是很重要的概念，实际真正干事的还是`ViewRoot`。![这里写图片描述](http://img.blog.csdn.net/20161023113352368)
`Activity`在创建的时候会调用`onAttach()`创建`PhoneWindow`这个类，并在`handleResumeActivity`时将窗口加入到`WindowManager`中，不过加入的实际上并不是窗口，而是`DecorView`。所以其实在客户端的核心概念只有`ViewRoot`，`DecorView`以及`ViewGroup`。其中后面两者主要还是`View`的概念，真正完成与`WindowManager`进行通信的还是`ViewRoot`这个家伙。

#### ViewRoot
`ViewRoot`的真正实现类是`ViewRootImpl`。`ViewRoot`通过与`WindowManager`进行通信完成`addView`以及消息下发。![这里写图片描述](http://img.blog.csdn.net/20161023115558450)
`ViewRoot`通过`IWindowSession`将窗口加入到`WindowManager`中。
![这里写图片描述](http://img.blog.csdn.net/20161023144048607)
`WindowManager`通过`IWindow`接口下发事件到`Activity`。
所以`ViewRoot`其实本质上是一个`Handler`，用于接收消息并处理消息。
`Activity`利用`getSystemService`来获取`WindowManagerImpl`实例，而这个实例实际上就是`WindowManager`在客户端本地的代理：
```java
wm=(WindowManagerImpl)context.getSystemService(Context.WINDOW_SERVICE);
```
之后再调用`addView`接口通过`WindowManagerImpl`将窗口添加到`WindowManager`中。在`addView`的过程中,`WindowManagerImpl`会建立起`View`,`Layout`,`ViewRoot`之间的对应关系，然后利用`IWindowSession`传递给`WindowManager`。
![这里写图片描述](http://img.blog.csdn.net/20161023144823917)

### Server端——`WindowManager`
`WindowManager`是服务端管理窗口的组件，它管理的是各个应用的顶级窗口，也即`DecorView`。将所有的窗口归置到一个统一的系统服务`WindowManagerService`管理是Android系统的设计思想，这样的机制并不难理解，系统总要有一个总管各个窗口的管家嘛，总不能任其自生自灭。`WindowManagerService`的主要工作包括：
Window Service大体上实现了如下的功能：，
>（1）Z-ordered的维护函数
  （2）输入法管理
  （3）AddWindow/RemoveWindow
  （4）Layerout
  （5）Token管理，AppToken
  （6）活动窗口管理（FocusWindow）
  （7）活动应用管理（FocusAPP）
  （8）转场动画
  （9）系统消息收集线程
  （10）系统消息分发线程

在服务端窗口对象叫作`WindowState`，Server端维护一个`mWindow`，其实就是一个按Z-order排序的窗口数组。`mWindowMap`用于记录`<Client：Binder,WindowState对象>`。
`WindowState`通过本地的`client`实例维护`IWindow`实例，同时利用该实例访问窗口。
#### FocusWindow活动窗口如何计算
原理其实很简单，首先找到前台应用，然后根据`mWindow`找到Z-order顺序中第一位次的窗口，该窗口就是活动窗口。
#### 为什么要提出Token这个概念
Token在本质上就是一个标示符，应用程序使用改标示符来找到该应用的窗口。`AppToken：<Token：IBinder，allWindows>`。通过Token就可以管理该应用的所有窗口。

### `WindowManager`消息与分发
下面再来说一下`WindowManager`的系统消息收集与分发过程。`WindowManagerService`在内部维护了一个`KeyQ`的消息队列，同时还有两个线程：
> 1.InputDeviceReader
> 2.InputDispatcherThread

`InputDeviceReader`使用Native函数`readEvent`从driver中读取`RawEvent`并放到`KeyQ`队列中。
`InputDispatherThread`负责从`KeyQ`队列中读取事件，并在`WindowManager`找到对应的窗口，利用该窗口的`IWindow`接口下发事件。