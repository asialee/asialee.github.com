---
layout: post
title: Android应用框架之Service
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
之前的博客已经介绍了应用框架中的`Activity`和`Application`,今天来讲四大组件之一的`Service`。对于`Service`大家肯定都比较熟悉，与`Activity`最大的不同就是`Service`不会与界面打交道，而是始终工作在后台，执行一些与UI无关的操作和计算。即便用户切换了其他应用，启动的Service仍可在后台运行。一个组件可以与Service绑定并与之交互，甚至是跨进程通信（IPC）。
Service运行在主线程中（A service runs in the main thread of its hosting process），Service并不是一个新的线程，也不是新的进程。也就是说，若您需要在Service中执行较为耗时的操作（如播放音乐、执行网络请求等），需要在Service中创建一个新的线程。这可以防止ANR的发生，同时主线程可以执行正常的UI操作。
`Service`有两种启动方式，一个是`startService`，一个是`bindService`,接下来分别介绍一下两种方式的启动逻辑。

### 1.`startService`
通常情况下启动一个`Service`的代码如下：
```
Intent intent = new Intent(this, MyService.class);
context.startService(intent);
```
启动过程是从`Context`开始的，而这个`Context`实际是一个`ContextWrapper`，而从`ContextWrapper`的实现看来，其内部实现都是通过`ContextImpl`来完成的，这是一种典型的桥接模式。通过调用`ContextImpl`的`startService`，会启动一个服务，核心代码如下所示：
```
private Component startServiceCommon(Intent service, UserHandle user) {
......
Component cn = ActivityManagerNative.getDefault().startService(mMainThread.getApplicationThread(),service,service.resolveTypeIfNeeded(getContentResolver()),user.getIdentifier());
......
```
`ContextImpl`通过`ActivityManagerNative.getDefault()`获取到一个服务，这个服务就是熟悉的`Activity Manager Service(AMS)`,启动这个服务的方式当然还是Binder机制。所起启动`Service`的工作就转移到了AMS身上。在AMS的内部还有一个`mServices`,这个对象是辅助AMS进行service管理的类，包括Service的启动、绑定和停止等等。同时一个Service在AMS内部对应一个`ServiceRecord`，AMS用它来记录各个Service。
而在AMS内部会通过`realStartServiceLocked`方法来启动Service，其实在AMS内部的启动步骤还有还经过了很多方法，不过最为核心的就是`realStartServiceLocked`，该方法的核心代码如下：
```java
private final void realStartServiceLocked(ServiceRecord r, ProcessRecord app, boolean execInFg) throws RemoteException {
......				    app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
app.thread.scheduleCreateService(r, r.serviceInfo, mMm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo), app.repProcState);
r.postNotification();
......
}
```
这里的app是一个`ProcessRecord`对象，就是在之前的博客中提到的AMS中用于记录一个Application的对象。通过`app.thread.scheduleCreateService`方法来创建`Service`并调用其`onCreate`方法，接着在通过`sendServiceArgsLocked`方法来调用`Service`的其他方法，比如`onStartCommand`。而这两个过程均是进程间通信，`app.thread`其实是一个`IApplicationThread`类型，实际就是一个`Binder`。而`scheduleCreateService`就是这个binder中的一个接口方法，接下来看一下对应的`scheduleCreateService`方法：
```
public final void scheduleCreateService(Binder token, ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
updateProcessState(procesState, false);
CreateServiceData s = new CreateServiceData();
s.token = token;
s.info = info;
s.compatInfo = compatInfo;

sendMessage(H.CREATE_SERVICE, s);
}
```
从代码中可以看到，最后的创建工作又通过发送消息给Handler H将创建Service的工作又回到了ActivityThread中。最后再来看看ActivityThread的`handleCreateService`
```
private void handleCreateService(CreateServiceData data){
......
Service service = null;
java.lang.ClassLoader cl = packageInfo.getClassLoader();
service = (Service)cl.loadClass(data.info.name).newInstance();
......
ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
context.setOuterContext(service);

Application app = packageInfo.makeApplication(false, mInstrumentation);
service.attach(context, this, data.info.name, data.token, app, ActivityManagerNative.getDefault());
service.onCreate();
mServices.put(data.token, service);
......
}
```
这个方法主要做了以下几件事：

> 1.通过创建类加载器，创建Service实例
> 2.创建Application对象，并调用其onCreate方法，当然Application对象只会被创建一次
> 3.创建ContextImpl对象，并通过service的onAttach方法建立两者之间的联系。这个过程和Activity类似，毕竟Activity和Service都是一个Context
> 4.最后调用Service的onCreate方法，并将Service保存在ActivityThread中的一个列表mServices。

由于Service的`onCreate`方法被执行了，接下来`AcitivtyThread`还会通过`handleServiceArgs`方法调用Service的onStartCommand方法：
```
private void handleServiceArgs(ServiceData data) {
Service s = mServices.get(data.token);
......
if(!data.taskRemoved) {
	res = s.onStartCommand(data.args, data.flags, data.startId);
} else {
	s.onTaskRemoved(data.args);
	res = Service.START_TASK_REMOVED_COMPLETE;
}
......
ActivityManagerNative.getDefault().serviceDoneExecuting(data.token, 1, data.startId, res);
......
}
```
在这个方法中可以看到，在`service`执行完成之后，还会通过`ActivityManagerNative.getDefault().serviceDoneExecuting`来通知AMS service已经执行完毕。
最后来总结一下Service启动的主要步骤：
> Context-->AMS-->app.thread-->ActivityThread-->Service

为什么要去绕这么一大圈呢？其实很好理解，AMS管理各个组件，要创建一个新的service当然要通过AMS来维护一个与该service对应的实例并与对应的进程实现关联，app.thread只是一个应用通信的接口，并将对应的工作交接给ActivityThread，ActivityThread才是应用的真正实例，它当然也要管理该Service，并维护一个对应的记录（mServices）。其实Activity和Service的启动过程大致相同，从中可以更加了解Android的应用框架。

### 2.`bindService`
`bindService`的大致过程过程和`startService`类似，还是通过`contextImpl.bindServiceCommon`来启动：
```
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, UserHandle user) {
IServiceConnction sd;
......
sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), mMainThread.getHandler(), flags);
......
int res = ActivityManagerNative.getDefault().bindService(mMainThread.getApplicationThread(), getActivityToken(), service, service.resolveTypeIfNeeded(getContentResolver()), sd, flags, user.getIdentifier());
......
}
```
这里主要做了两件事：

- 将客户端的ServiceConnection转化为ServiceDispatcher.InnerConnection。之所以不能直接使用ServiceConnection是因为绑定的服务可能是跨进程的，所以必须借助于Binder才能让远程服务回调自己的方法。而ServiceDispatcher的内部类InnerConnction正好充当了这个Binder。所以ServiceDispatcher的作用就是ServiceConnection和InnerConnection连接的桥梁。

- 调用AMS的bindService方法来完成Service的具体绑定过程。

接下来重点讲一下AMS的bindService方法。和`startService`方法类似的是，`bindService`最终会将调用到`app.thread.scheduleBindService()`:
```
public final void scheduleBindService(Binder token, Intent intent, boolean rebind, int processState) {
	updateProcessState(processState, false);
	BindServiceData s = new BindServiceData();
	s.token = token;
	s.intent = intent;
	s.rebind = rebind;
	......
	sendMessage(H.BIND_SERVICE, s);
}
```
接下来又转移到了`ActivityThread`中，而这个方法就是`ActivityThread.handleBindService()`：
```
private void handleBindService(BindServiceData data) {
	Service s = mServices.get(data.token);
	......
	IBiner binder = s.onBind(data.intent);				 ActivityManagerNative.getDefault().publishService(data.token, data.intent, binder);
	......
}
```
在`handleBindService`中，首先根据Service的token取出Service对象，然后调用Service的onBind方法。但是onBind方法是Service的方法，这个时候客户端并不知道已经绑定成功了，所以还必须调用客户端的ServiceConnection中的onServiceConnected，这个是由`ActivityManagerNative.getDefault().publishService`方法来完成的。最终指令流会转移到mServices（AMS内部的辅助Service）的publishServiceLocked。其核心代码只有一行：`c.conn.connected(r.name, service)`,其中c.conn类型是ServiceDispatcher.InnerConnection，service就是Service的onBind返回的Binder对象。接下来看看ServiceDispatcher.InnerConnection的定义：
```
private static class InnerConnection extends IServiceConnection.Stub {
	...
	private void connected(ComponentName name, Binder service) throws RemoteException {
		LoadedApk.ServiceDispatcher sd = mDispacher.get();
		if(sd != null) {
			sd.connectd(name, service);
		}
	}
}
```
InnerConnection最后通过ServiceDispatcher的connected方法来调用ServiceConnection的onServiceConnected，至此绑定完成。