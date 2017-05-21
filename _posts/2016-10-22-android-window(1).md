---
layout: post
title: Android窗口管理（1）——窗口基本架构
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
本文给大家介绍以下Android窗口的基本架构，平时我们在编码时打交道最多的就是各种View以及各种Layout。但系统窗口究竟是以何种形式将这些组件组织在一起，在View和Layout的上层又是通过哪些组件以什么样的方式来管理的？首先来看一下Window的基本结构：![这里写图片描述](http://img.blog.csdn.net/20161023151515419)
从图中可以看到，除了`ViewGroup`和`View`这些非常熟悉的组件了，在其之上还有`ViewRoot`、`DecorView`、`PhoneWindow`三个‘管家’来管理这些小弟。那么它们究竟分别有什么作用呢？

### `PhoneWindow`
要想知道`PhoneWindow`具体是做什么的，首先需要了解一下`Window`具体是什么：
>Abstract base class for a top-level window look and behavior policy. An instance of this class should be used as the top-level view added to the window manager. It provides standard UI policies such as a background, title area, default key processing, etc.

以上是官方文档对于`Window`的介绍，从中可以知道Window是一个管理视图的顶层容器，负责视图的绘制以及行为，包括标题，背景等等都归其负责。该类的实例也会被添加到WindowManager加以管理。看起来`Window`的权利很大，但其实它只是一个抽象类，说白了就是一个傀儡，具体的工作都不是它做的，而是交给了我们要讲的`PhoneWindow`。
`PhoneWindow`是`Window`唯一的实现类，也继承了`Window`所有的权利，可以说是一个挟天子以令诸侯的狠角色。不过我们平时很少会直接与`PhoneWindow`打交道，日常的实物都是交给`ViewRoot`去打理的。
通过上述的介绍，我们可以发现`Window`在Android系统中并不是一个很重要的概念，这似乎是Google有意为之的，弱化`Window`概念而强化`View`概念。

### `ViewRoot`
`ViewRoot`对应于`ViewRootImpl`，它是连接`WindowManager`和`DecorView`的纽带，View的三大流程（measure，layout，draw，后续文章会介绍这三大流程）均是通过`ViewRootImpl`来完成的。在`ActivityThread`（`ActivityThread`是应用程序空间中的中重要概念，真正对应应用进程的不是`Application`而是`ActivityThread`。每个应用程序都是以`ActivityThread.main()`作为程序入口进入到消息循环处理中并提供了一个`IActivityThread`接口作为与`Activity Manager Service`的通讯接口.通过该接口AMS可以将`Activity`的状态变化传递到客户端的`Activity`对象，详细内容后续文章会介绍）中，当`Activity`对象被创建完毕后，会将`DecorView`添加到`Window`中，同时会创建`ViewRootImpl`对象，并为两者建立关联。

### `DecorView`
虽然`PhoneWindow`,`ViewRoot`都是属于Android窗口结构中的一部分，但是真正与用户所见相关的部分还是从`DecorView`开始的。`DecorView`才是真正的顶级`View`，一般情况下它内部会包含一个竖直方向的`LinearLayout`，在这个`LinearLayout`里面有上下两个部分，上面是标题栏，下方才是内容栏。在`Activity`中通过我们通过设置`setContentView`设置的部分就是内容栏部分。内容栏的id为content，所以称之为`setContentView`,通过代码：
```
ViewGroup content = (ViewGroup)findViewById(R.id.content)
```
即可以得到content。而content内部便是我们自己设定的View了。
以上内容便是Android窗口的基本架构，理解清楚了基本结构后，才能更好地理解View的事件分发以及和WindowManager的交互。这部分内容会在下一篇博客中详细介绍。
