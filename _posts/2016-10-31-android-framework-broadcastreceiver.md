---
layout: post
title: Android应用框架之BroadcastReceiver
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
广播机制是Android系统中的一种消息传播机制，通过观察者模式实现了消息发送者与消息接收者之间的解耦。`BroadcastReceiver`的使用方式有两种，一种是静态注册，即在Manifest文件中注册，然后在需要发送广播时调用`context.sendBroadcast(intent)；`；第二种是动态注册。`BroadcastReceiver`的使用不是本文的重点，本文将着重讲解广播的注册过程和消息发送及接收过程。

### 1 广播注册过程
广播的静态注册是通过PMS(PackageManagerService)来完成的，其余三大组件也是通过PMS来完成注册的。这里重点讲一下`BroadcastReceiver`的动态启动方法。和`Activity`以及`Service`一样，其启动过程也是通过`ContextWrapper-->ContextImpl`来完成的。其主要的启动函数是`ContextImpl.registerReceiver`:
```
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId, IntentFilter filter, String broadcastPermission, Handler scheduler, Context context) {
	IIntentReceiver rd = null;
	......
	rd = mPackageInfo.getReceiverDispatcher(receiver, context, scheduler, mMainThread.getInstrumentation, true);
	......
	return ActivityManagerNatvice.getDefault().registerReceiver(mMainThread.getApplicationThread(), mBasePackageName, rd, filter, boradcastPermission, userId);
	......
}
```
从上面的代码可以看出主要做了两件事：

- 从`mPackageInfo`获取`IIntentReceiver`对象，之所以这样和`bindService`是一样的，因为上述的注册过程是一个跨进程的通信方式，而`BroadcastReceiver`作为Android的一个组件是不能直接跨进程传递的，所以需要使用`IIntentReceiver`来中转。其具体是由`LoadedApk.ReceiverDispatcher.InnerReceiver`，`ReceiverDispatcher`内部同时保存了`BroadcastReceiver`和`InnerReceiver`，所以当接收到广播时，`ReceiverDispatcher`可以很方便地调用`BroadcastReceiver.onReceive()`方法。

- 通过`ActivityManagerNative.getDefault()`获取`ActivityManagerService`,然后通过AMS来完成广播的注册过程。

接下来具体看一下AMS的`registerReceiver`具体的实现:
```
public Intent registerReceiver(IApplicationThread caller, String callerPackage, IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
	......
	mRegisterReceivers.put(receiver.asBinder(), rl);
	
	BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage, permission, callingUid, userId);
	rl.add(bf);
	mReceiverResolver.addFilter(bf);
}
```

### 2 广播的发送和接收过程
广播的发送通过`ContextImpl.sendBroadcast`方法：
```
public void sendBroadcast(Intent intent) {
	......
	ActivityManagerNative.getDefault().broadcastIntent(mMainThread.getApplicationThread(), intent, resolvedType, null, Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, false, false, getUserId());
	......
}
```
不出意料，任务又转到了AMS中，AMS在接收到这个指令会调用内部的`broadcastIntentLocked`方法，在该方法中，AMS会根据intent-filter查找出匹配的广播接收者，并通过一系列的条件过滤，并将最终满足条件的广播接收者添加到`BroadcastQueue`中，然后`BroadcastQueue`会将广播发送到相应的广播接收者,核心代码如下：
```
BroadcastQueue queue = broadcastQueueForIntent(intent);
BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp, callerPackage, callingPid, callingUid, resolvedType, requiredPermission, appOp, receivers, resultTo, resultCode, resultData, map, ordered, sticky, false, userId);
......
queue.enqueOrderedBroadcastLocked(r);
queue.scheduleBroadcastsLocked();
```
下面再看一下在`BroadcastQueue`中发送广播`scheduleBroadcastsLocked`的实现：
```
public void scheduleBroadcastsLocked(){
	......
	mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
	......
}
```
实际上`BroadcastQueue`的`scheduleBroadcastsLocked`方法没有立即发送广播，而是发送了一个BROADCAST_INTENT_MSG类型的消息，BroadcastQueue收到该消息后会调用`processNestBroadcast`方法：
```
while(mParallelBroadcasts.size() > 0) {
	r = mParallelBroadcasts.remove(0);
	r.dispatchTime = SystemClock.uptimeMillis();
	r.dispatchClockTime = System.currentTimeMillis();
	final int N = r.recivers.size();
	for(int i = 0; i < N; i++) {
		Object target = r.receivers.get(i);
		deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
	}
	addBroadcastToHistoryLocked(r);
}
```
可以看到无序广播存储在mParallelBroadcasts中，系统遍历该队列，并将广播发送给它所有的接收者。具体的发送工作通过`deliverToRegisteredReceiverLocked`完成，在该函数内部通过`performReceivedLocked`来完成：
```
private static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver, Intent intent, int resultCode, String data, Bundle extras, boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
	......
	app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode, data, extras, ordered, sticky, sendingUser, app.repProcState);
	......
}
```
ApplicationThread的scheduleRegisteredReceiver会调用InnerReceiver.performReceive来实现广播的接收。而在这个方法中会调用LoadedApk.ReceiverDispatcher.performReceive方法：
```
public void performReceive(Intent intent, int resultCode, String data, Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
	......
	Args args = new Args(intent, resultCode, data, extras, ordered, sticky, sendingUser);
	if(!mActivityThread.post(args)) {
		if(mRistered && ordered) {
			IActivityManager mgr = ActivityManagerNative.getDefault();
			args.sendFinished(mgr);
		}
	}
}
```
在上面的代码中，会创建一个Args对象，并通过mActivityThread的post方法来执行Args的逻辑，Args实际上是一个Runnable接口。mActivityThread是一个Handler，其实就是ActivityThread中的mH，类型是H。Args中的run方法有如下几行代码：
```
final BroadcastReceiver receiver = mReceiver;
receiver.setPendingResult(this);
receiver.onReceive(mContext, intent);
```
这个时候BroadcastReceiver的onReceive方法才被执行，也就接收到广播了。