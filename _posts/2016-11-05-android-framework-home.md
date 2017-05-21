---
layout: post
title: Android应用框架之Home程序（Launcher）
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
上一篇博客我们讲了`PackageManagerService`的启动过程以及对于应用程序的注册过程，当系统启动完成后，系统需要开启第一个应用程序，这就是Home程序，也就是我们熟知的桌面程序。本篇博客主要介绍Home的启动过程。
通过上一篇博客介绍，我们知道系统在启动的时候会启动`SystemServer`,并且在`SystemServer`中会启动一系列的Service，包括`PackageManagerService`,`ActivityManagerService`等等，而`ActivityManagerService`在启动后就会负责Home的启动。所以一开始先来看看`ActivityManagerService的启动`。

### 1.`ActivityManagerService`
通过前一篇博客的介绍，我们知道`SystemServer`在启动后会开启一个线程`ServerThread`来启动各种系统级Service，其中就包括`ActivityManagerService`,`ServerThread`的run函数可以参看上一篇博客。接下来看看`ActivityManagerService`启动后都干些什么。
#### 1 `ActivityManagerService.main`
```
public final class ActivityManagerService extends ActivityManagerNative
		implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
	......
	public static final Context main(int factoryTest) {
		AThread thr = new AThread();
		thr.start();

		synchronized (thr) {
			while (thr.mService == null) {
				try {
					thr.wait();
				} catch (InterruptedException e) {
				}
			}
		}

		ActivityManagerService m = thr.mService;
		mSelf = m;
		ActivityThread at = ActivityThread.systemMain();
		mSystemThread = at;
		Context context = at.getSystemContext();
		m.mContext = context;
		m.mFactoryTest = factoryTest;
		m.mMainStack = new ActivityStack(m, context, true);

		m.mBatteryStatsService.publish(context);
		m.mUsageStatsService.publish(context);

		synchronized (thr) {
			thr.mReady = true;
			thr.notifyAll();
		}

		m.startRunning(null, null, null, null);
		return context;
	}
	......
}
```
这个函数首先通过AThread线程对象来内部创建了一个ActivityManagerService实例，然后将这个实例保存其成员变量mService中，接着又把这个ActivityManagerService实例保存在ActivityManagerService类的静态成员变量mSelf中，最后初始化其它成员变量，就结束了。

#### 2.`ActivityManagerService.setProcessSystem`
```
public final class ActivityManagerService extends ActivityManagerNative
		implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
	......

	public static void setSystemProcess() {
		try {
			ActivityManagerService m = mSelf;

			ServiceManager.addService("activity", m);
			ServiceManager.addService("meminfo", new MemBinder(m));
			if (MONITOR_CPU_USAGE) {
				ServiceManager.addService("cpuinfo", new CpuBinder(m));
			}
			ServiceManager.addService("permission", new PermissionController(m));

			ApplicationInfo info =
				mSelf.mContext.getPackageManager().getApplicationInfo(
				"android", STOCK_PM_FLAGS);
			mSystemThread.installSystemApplicationInfo(info);

			synchronized (mSelf) {
				ProcessRecord app = mSelf.newProcessRecordLocked(
					mSystemThread.getApplicationThread(), info,
					info.processName);
				app.persistent = true;
				app.pid = MY_PID;
				app.maxAdj = SYSTEM_ADJ;
				mSelf.mProcessNames.put(app.processName, app.info.uid, app);
				synchronized (mSelf.mPidsSelfLocked) {
					mSelf.mPidsSelfLocked.put(app.pid, app);
				}
				mSelf.updateLruProcessLocked(app, true, true);
			}
		} catch (PackageManager.NameNotFoundException e) {
			throw new RuntimeException(
				"Unable to find android system package", e);
		}
	}
	......
}
```
这个函数主要做了两件事，第一是将`ActivityManagerService`实例添加到`ServiceManager`中，这样其他组件就可以通过`getSystemService`接口来获取到`ActivityManagerSerivce`了。
第二件事就是通过调用mSystemThread.installSystemApplicationInfo函数来把应用程序框架层下面的android包加载进来。

#### 3.`ActivityManagerService.systemReady`
接下来`ActivityManagerService`会调用`systemReadey`接口。
```
public final class ActivityManagerService extends ActivityManagerNative
		implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
	......
	public void systemReady(final Runnable goingCallback) {
		......
		synchronized (this) {
			......
			mMainStack.resumeTopActivityLocked(null);
		}
	}
	......
}
```
这里就是通过mMainStack.resumeTopActivityLocked函数来启动Home应用程序的了，这里的mMainStack是一个ActivityStack类型的实例变量。

### 2.Home应用程序启动

#### 1.`mMainStack.resumeTopActivityLocked`
接着上面的指令流继续往下看，前文已经讲到了，`mMainStack`是一个ActivityStack，即一个activity的栈，每个应用程序都会有一个或多个`ActivityStack`用来维护activity。而`resumeTopActivityLocked`就是把栈顶的activity恢复到前台。
```
public class ActivityStack {
	......

	final boolean resumeTopActivityLocked(ActivityRecord prev) {
		// Find the first activity that is not finishing.
		ActivityRecord next = topRunningActivityLocked(null);
		......
		if (next == null) {
			// There are no more activities!  Let's just start up the
			// Launcher...
			if (mMainStack) {
				return mService.startHomeActivityLocked();
			}
		}
		......
	}
	......
}
```
由于当前Home应用程序并没有启动，所以next为null，进而会调用`mService.startHomeActivityLocked`来启动Home程序。

#### 2.`mService.startHomeActivityLocked`
```
public final class ActivityManagerService extends ActivityManagerNative
		implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
	......
	boolean startHomeActivityLocked() {
		......
		Intent intent = new Intent(
			mTopAction,
			mTopData != null ? Uri.parse(mTopData) : null);
		intent.setComponent(mTopComponent);
		if (mFactoryTest != SystemServer.FACTORY_TEST_LOW_LEVEL) {
			intent.addCategory(Intent.CATEGORY_HOME);
		}
		ActivityInfo aInfo =
			intent.resolveActivityInfo(mContext.getPackageManager(),
			STOCK_PM_FLAGS);
		if (aInfo != null) {
			intent.setComponent(new ComponentName(
				aInfo.applicationInfo.packageName, aInfo.name));
			// Don't do this if the home app is currently being
			// instrumented.
			ProcessRecord app = getProcessRecordLocked(aInfo.processName,
				aInfo.applicationInfo.uid);
			if (app == null || app.instrumentationClass == null) {
				intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
				mMainStack.startActivityLocked(null, intent, null, null, 0, aInfo,
					null, null, 0, 0, 0, false, false);
			}
		}
		return true;
	}
	......
}
```
在这个函数中，可以看到AMS会创建一个intent实例，并且设置其category为HOME。而category为Home当然就是Home程序的启动Activity。接下来通过resolveActivityInfo向`PackageManagerService`查询对应的Activity，当然就是Home程序的`Activity`了。接下来通过`mMainStack.startActivityLocked`就启动了Home程序。


#### 3.`Launcher.onCreate`
```
public final class Launcher extends Activity
		implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks, AllAppsView.Watcher {
	......
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		......
		if (!mRestoring) {
			mModel.startLoader(this, true);
		}
		......
	}
	......
}

public class LauncherModel extends BroadcastReceiver {
	......

	public void startLoader(Context context, boolean isLaunching) {
		......

                synchronized (mLock) {
                     ......

                     // Don't bother to start the thread if we know it's not going to do anything
                     if (mCallbacks != null && mCallbacks.get() != null) {
                         // If there is already one running, tell it to stop.
                         LoaderTask oldTask = mLoaderTask;
                         if (oldTask != null) {
                             if (oldTask.isLaunching()) {
                                 // don't downgrade isLaunching if we're already running
                                 isLaunching = true;
                             }
                             oldTask.stopLocked();
		         }
		         mLoaderTask = new LoaderTask(context, isLaunching);
		         sWorker.post(mLoaderTask);
	            }
	       }
	}

	......
}
```
通过上面两段代码可以发现，在Launcher的onCreate初始化函数中，通过mModel来加载Loader，这里的mModel是一个LauncherModel类型的成员变量。 这里不是直接加载应用程序，而是把加载应用程序的操作作为一个消息来处理。这里的sWorker是一个Handler，通过它的post方式把一个消息放在消息队列中去，然后系统就会调用传进去的参数mLoaderTask的run函数来处理这个消息，这个mLoaderTask是LoaderTask类型的实例，于是，下面就会执行LoaderTask类的run函数了。

#### 4.` LoaderTask.loadAllAppsByBatch`
在LoaderTask的run函数会调用`loadAndBindAllApps`,而在这个函数中又会调用`loadAllAppsByBatch`,所以真正启动各个app的工作是在这个函数中执行的。
```
public class LauncherModel extends BroadcastReceiver {
	......
	private class LoaderTask implements Runnable {
		......
		private void loadAllAppsByBatch() {	
			......
			final Intent mainIntent = new Intent(Intent.ACTION_MAIN, null);
			mainIntent.addCategory(Intent.CATEGORY_LAUNCHER);

			final PackageManager packageManager = mContext.getPackageManager();
			List<ResolveInfo> apps = null;

			int N = Integer.MAX_VALUE;

			int startIndex;
			int i=0;
			int batchSize = -1;
			while (i < N && !mStopped) {
				if (i == 0) {
					mAllAppsList.clear();
					......
					apps = packageManager.queryIntentActivities(mainIntent, 0);
					......
					N = apps.size();			
					......
					if (mBatchSize == 0) {
						batchSize = N;
					} else {
						batchSize = mBatchSize;
					}
					......
					Collections.sort(apps,
						new ResolveInfo.DisplayNameComparator(packageManager));
				}
				startIndex = i;
				for (int j=0; i<N && j<batchSize; j++) {
					// This builds the icon bitmaps.
					mAllAppsList.add(new ApplicationInfo(apps.get(i), mIconCache));
					i++;
				}

				final boolean first = i <= batchSize;
				final Callbacks callbacks = tryGetCallbacks(oldCallbacks);
				final ArrayList<ApplicationInfo> added = mAllAppsList.added;
				mAllAppsList.added = new ArrayList<ApplicationInfo>();
			
				mHandler.post(new Runnable() {
					public void run() {
						final long t = SystemClock.uptimeMillis();
						if (callbacks != null) {
							if (first) {
								callbacks.bindAllApplications(added);
							} else {
								callbacks.bindAppsAdded(added);
							}
							......
						} else {
							......
						}
					}
				});
				......
			}
			......
		}
		......
	}
	......
}
```
函数首先构造一个CATEGORY_LAUNCHER类型的Intent,接着从mContext变量中获得`PackageManagerService`的接口。下一步就是通过这个`PackageManagerService.queryIntentActivities`接口来取回所有Action类型为Intent.ACTION_MAIN，并且Category类型为Intent.CATEGORY_LAUNCHER的Activity了。从queryIntentActivities函数调用处返回所要求的Activity后，便调用函数tryGetCallbacks(oldCallbacks)得到一个返CallBack接口，这个接口是由Launcher类实现的，接着调用这个接口的.bindAllApplications函数来进一步操作。注意，这里又是通过消息来处理加载应用程序的操作的。

#### 5.` Launcher.bindAllApplications`
```
public final class Launcher extends Activity
		implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks, AllAppsView.Watcher {
	......
	private AllAppsView mAllAppsGrid;
	......
	public void bindAllApplications(ArrayList<ApplicationInfo> apps) {
		mAllAppsGrid.setApps(apps);
	}
	......
}
```
函数很简单，就是调用了mAllAppsGrid.setApps(apps)。这里的mAllAppsGrid是一个AllAppsView类型的变量，它的实际类型一般就是AllApps2D了。所以这个函数的作用很清晰了，就是在Home界面绘制各个应用的图标。具体的绘制逻辑这里就不多讲了。
到这里Home应用程序的启动过程就介绍完了。虽然函数的调用过程比较复杂，但其实总的逻辑还是比较简单的:
> 1. 创建ActivityManagerService；
> 2. ActivityManagerService通过mMainStack来启动Home程序
> 3. mMainStack向PackageManagerService查询Home程序的Activity，然后启动该Activity，并放入该mMainStack中
> 4. Home程序启动后通过PackageManagerService查询所有应用程序的启动Activity
> 5. Home程序在初始化的时候绘制Home界面
> 6. 当点击某个应用程序图标的时候，启动对应应用程序的启动Activity  