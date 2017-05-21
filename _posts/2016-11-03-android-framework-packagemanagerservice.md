---
layout: post
title: Android应用框架之PackageManagerService
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
系统在启动的时候会启动一个叫做`PackageManagerService`的服务，顾名思义，这个服务主要管理安装在设备上的应用程序，其中最为重要的工作就是在在系统启动之后，`PackageManagerService`会扫描特定目录下地以apk为后缀的文件，然后将对应的应用安装到系统中。注意，这里的安装并不是我们平时所说的安装，它指的的是将存放在磁盘之上的静态应用程序文件进行解析，并将相关信息注册到系统中。而具体的解析工作实际就是读取应用的配置文件`manifest.xml`，并将文件中配置的组件
（`Activity`,`Service`,`BroadcastRecevier`,`ContentProvider`），权限等信息注册到`PackageManagerService`中。
本篇博客主要介绍`PackageManagerService`的启动过程，以及`PackageManagerService`如何安装各个应用程序。

### 1.`PackageManagerService`启动过程
和`ActivityManagerService`，`WindowManagerService`一样，`PackageManagerService`是一个系统级的服务，运行在独立的进程中，而所有的系统级服务都是由`SystemServer`启动的。所以首先来看看`SystemServer`的启动过程。

##### 1) `SystemServer`启动：
SystemServer组件是由Zygote进程负责启动的，启动的时候就会调用它的main函数，这个函数主要调用了JNI方法init1来做一些系统初始化的工作。
```
public class SystemServer
{
	......
	native public static void init1(String[] args);
	......
	public static void main(String[] args) {
		......
		init1(args);
		......
	}
	......
}
```
##### 2)`SystemServer.system_init`
经过一系列调用后转到`system_init`方法，这是一个JNI方法
```
extern "C" status_t system_init()
{
	LOGI("Entered system_init()");
	sp<ProcessState> proc(ProcessState::self());
	sp<IServiceManager> sm = defaultServiceManager();
	LOGI("ServiceManager: %p\n", sm.get());
	sp<GrimReaper> grim = new GrimReaper();
	sm->asBinder()->linkToDeath(grim, grim.get(), 0);
	char propBuf[PROPERTY_VALUE_MAX];
	property_get("system_init.startsurfaceflinger", propBuf, "1");
	if (strcmp(propBuf, "1") == 0) {
		// Start the SurfaceFlinger
		SurfaceFlinger::instantiate();
	}

	// Start the sensor service
	SensorService::instantiate();

	// On the simulator, audioflinger et al don't get started the
	// same way as on the device, and we need to start them here
	if (!proc->supportsProcesses()) {

		// Start the AudioFlinger
		AudioFlinger::instantiate();

		// Start the media playback service
		MediaPlayerService::instantiate();

		// Start the camera service
		CameraService::instantiate();

		// Start the audio policy service
		AudioPolicyService::instantiate();
	}

	// And now start the Android runtime.  We have to do this bit
	// of nastiness because the Android runtime initialization requires
	// some of the core system services to already be started.
	// All other servers should just start the Android runtime at
	// the beginning of their processes's main(), before calling
	// the init function.
	LOGI("System server: starting Android runtime.\n");

	AndroidRuntime* runtime = AndroidRuntime::getRuntime();

	LOGI("System server: starting Android services.\n");
	runtime->callStatic("com/android/server/SystemServer", "init2");

	// If running in our own process, just go into the thread
	// pool.  Otherwise, call the initialization finished
	// func to let this process continue its initilization.
	if (proc->supportsProcesses()) {
		LOGI("System server: entering thread pool.\n");
		ProcessState::self()->startThreadPool();
		IPCThreadState::self()->joinThreadPool();
		LOGI("System server: exiting thread pool.\n");
	}

	return NO_ERROR;
}
```
在这个方法中，创建了SurfaceFlinger、SensorService、AudioFlinger、MediaPlayerService、CameraService和AudioPolicyService这几个服务，然后就通过系统全局唯一的AndroidRuntime实例变量runtime的callStatic来调用SystemServer的init2函数了。init2函数很简单，创建一个线程，而`PackageManagerService`就是在这个线程中创建的。
```
public class SystemServer
{
	......
	public static final void init2() {
		Slog.i(TAG, "Entered the Android system server!");
		Thread thr = new ServerThread();
		thr.setName("android.server.ServerThread");
		thr.start();
	}
}
```
##### 3)ServerThread.run
```
class ServerThread extends Thread {
	......
	@Override
	public void run() {
		......
		IPackageManager pm = null;
		......
		// Critical services...
		try {
			......
			Slog.i(TAG, "Package Manager");
			pm = PackageManagerService.main(context,
						factoryTest != SystemServer.FACTORY_TEST_OFF);
			......
		} catch (RuntimeException e) {
			Slog.e("System", "Failure starting core service", e);
		}
		......
	}
	......
}
```
在这个线程中创建了`PackageManagerService`,并同时启动了其main函数。另外在这个线程中还启动了`ActivityManagerService`等其他Service

### 2.应用安装
接下来再来看看`PackageManagerService`启动之后如何进行应用程序的安装。
###### 1）`PackageManagerService.main`
```
class PackageManagerService extends IPackageManager.Stub {
	......
	public static final IPackageManager main(Context context, boolean factoryTest) {
		PackageManagerService m = new PackageManagerService(context, factoryTest);
		ServiceManager.addService("package", m);
		return m;
	}
	......
}
```
可以看到，创建完成后，就加载到`ServiceManager`中。接下来看看`PackageManagerService`的构造函数：
```
class PackageManagerService extends IPackageManager.Stub {
	......

	public PackageManagerService(Context context, boolean factoryTest) {
		......

		synchronized (mInstallLock) {
			synchronized (mPackages) {
				......

				File dataDir = Environment.getDataDirectory();
				mAppDataDir = new File(dataDir, "data");
				mSecureAppDataDir = new File(dataDir, "secure/data");
				mDrmAppPrivateInstallDir = new File(dataDir, "app-private");

				......

				mFrameworkDir = new File(Environment.getRootDirectory(), "framework");
				mDalvikCacheDir = new File(dataDir, "dalvik-cache");

				......

				// Find base frameworks (resource packages without code).
				mFrameworkInstallObserver = new AppDirObserver(
				mFrameworkDir.getPath(), OBSERVER_EVENTS, true);
				mFrameworkInstallObserver.startWatching();
				scanDirLI(mFrameworkDir, PackageParser.PARSE_IS_SYSTEM
					| PackageParser.PARSE_IS_SYSTEM_DIR,
					scanMode | SCAN_NO_DEX, 0);

				// Collect all system packages.
				mSystemAppDir = new File(Environment.getRootDirectory(), "app");
				mSystemInstallObserver = new AppDirObserver(
					mSystemAppDir.getPath(), OBSERVER_EVENTS, true);
				mSystemInstallObserver.startWatching();
				scanDirLI(mSystemAppDir, PackageParser.PARSE_IS_SYSTEM
					| PackageParser.PARSE_IS_SYSTEM_DIR, scanMode, 0);

				// Collect all vendor packages.
				mVendorAppDir = new File("/vendor/app");
				mVendorInstallObserver = new AppDirObserver(
					mVendorAppDir.getPath(), OBSERVER_EVENTS, true);
				mVendorInstallObserver.startWatching();
				scanDirLI(mVendorAppDir, PackageParser.PARSE_IS_SYSTEM
					| PackageParser.PARSE_IS_SYSTEM_DIR, scanMode, 0);


				mAppInstallObserver = new AppDirObserver(
					mAppInstallDir.getPath(), OBSERVER_EVENTS, false);
				mAppInstallObserver.startWatching();
				scanDirLI(mAppInstallDir, 0, scanMode, 0);

				mDrmAppInstallObserver = new AppDirObserver(
					mDrmAppPrivateInstallDir.getPath(), OBSERVER_EVENTS, false);
				mDrmAppInstallObserver.startWatching();
				scanDirLI(mDrmAppPrivateInstallDir, PackageParser.PARSE_FORWARD_LOCK,
					scanMode, 0);

				......
			}
		}
	}

	......
}
```
可以看到，在构造函数中，`PackageManagerService`(PMS)会扫描特定目录下的APK文件，然后进行相关的加载工作，这些目录包括：
>  /system/framework
>  /system/app
>  /vendor/app
>  /data/app
>  /data/app-private

在每个路径下，都调用了`scanDirLI`函数，接下来看看对应的函数做了些什么。

##### 2）` PackageParser.parsePackage`
`scanDirLI`中又经过多次调用，具体就是扫描对应目录的文件，如果是apk文件，就找到apk文件中的manifest文件，最后再为每一个apk创建一个`PackageParser`对象，并将manifest文件传递给`PackageParser.parsePackage`。
```
public class PackageParser {
	......

	private Package parsePackage(
			Resources res, XmlResourceParser parser, int flags, String[] outError)
			throws XmlPullParserException, IOException {
		......

		String pkgName = parsePackageName(parser, attrs, flags, outError);
		
		......

		final Package pkg = new Package(pkgName);

		......

		int type;

		......
		
		TypedArray sa = res.obtainAttributes(attrs,
			com.android.internal.R.styleable.AndroidManifest);

		......

		while ((type=parser.next()) != parser.END_DOCUMENT
			&& (type != parser.END_TAG || parser.getDepth() > outerDepth)) {
				if (type == parser.END_TAG || type == parser.TEXT) {
					continue;
				}

				String tagName = parser.getName();
				if (tagName.equals("application")) {
					......

					if (!parseApplication(pkg, res, parser, attrs, flags, outError)) {
						return null;
					}
				} else if (tagName.equals("permission-group")) {
					......
				} else if (tagName.equals("permission")) {
					......
				} else if (tagName.equals("permission-tree")) {
					......
				} else if (tagName.equals("uses-permission")) {
					......
				} else if (tagName.equals("uses-configuration")) {
					......
				} else if (tagName.equals("uses-feature")) {
					......
				} else if (tagName.equals("uses-sdk")) {
					......
				} else if (tagName.equals("supports-screens")) {
					......
				} else if (tagName.equals("protected-broadcast")) {
					......
				} else if (tagName.equals("instrumentation")) {
					......
				} else if (tagName.equals("original-package")) {
					......
				} else if (tagName.equals("adopt-permissions")) {
					......
				} else if (tagName.equals("uses-gl-texture")) {
					......
				} else if (tagName.equals("compatible-screens")) {
					......
				} else if (tagName.equals("eat-comment")) {
					......
				} else if (RIGID_PARSER) {
					......
				} else {
					......
				}
		}

		......

		return pkg;
	}

	......
		private Package parsePackage(
			Resources res, XmlResourceParser parser, int flags, String[] outError)
			throws XmlPullParserException, IOException {
		......

		String pkgName = parsePackageName(parser, attrs, flags, outError);
		
		......

		final Package pkg = new Package(pkgName);

		......

		int type;

		......
		
		TypedArray sa = res.obtainAttributes(attrs,
			com.android.internal.R.styleable.AndroidManifest);

		......

		while ((type=parser.next()) != parser.END_DOCUMENT
			&& (type != parser.END_TAG || parser.getDepth() > outerDepth)) {
				if (type == parser.END_TAG || type == parser.TEXT) {
					continue;
				}

				String tagName = parser.getName();
				if (tagName.equals("application")) {
					......

					if (!parseApplication(pkg, res, parser, attrs, flags, outError)) {
						return null;
					}
				} else if (tagName.equals("permission-group")) {
					......
				} else if (tagName.equals("permission")) {
					......
				} else if (tagName.equals("permission-tree")) {
					......
				} else if (tagName.equals("uses-permission")) {
					......
				} else if (tagName.equals("uses-configuration")) {
					......
				} else if (tagName.equals("uses-feature")) {
					......
				} else if (tagName.equals("uses-sdk")) {
					......
				} else if (tagName.equals("supports-screens")) {
					......
				} else if (tagName.equals("protected-broadcast")) {
					......
				} else if (tagName.equals("instrumentation")) {
					......
				} else if (tagName.equals("original-package")) {
					......
				} else if (tagName.equals("adopt-permissions")) {
					......
				} else if (tagName.equals("uses-gl-texture")) {
					......
				} else if (tagName.equals("compatible-screens")) {
					......
				} else if (tagName.equals("eat-comment")) {
					......
				} else if (RIGID_PARSER) {
					......
				} else {
					......
				}
		}

		......

		return pkg;
	}

	......
}
```
这里就是对AndroidManifest.xml文件中的application标签进行解析了，我们常用到的标签就有activity、service、receiver和provider，这里解析完成后，一层层返回，调用另一个版本的scanPackageLI函数把来解析后得到的应用程序信息保存下来。
```
class PackageManagerService extends IPackageManager.Stub {
	......

	// Keys are String (package name), values are Package.  This also serves
	// as the lock for the global state.  Methods that must be called with
	// this lock held have the prefix "LP".
	final HashMap<String, PackageParser.Package> mPackages =
		new HashMap<String, PackageParser.Package>();

	......

	// All available activities, for your resolving pleasure.
	final ActivityIntentResolver mActivities =
	new ActivityIntentResolver();

	// All available receivers, for your resolving pleasure.
	final ActivityIntentResolver mReceivers =
		new ActivityIntentResolver();

	// All available services, for your resolving pleasure.
	final ServiceIntentResolver mServices = new ServiceIntentResolver();

	// Keys are String (provider class name), values are Provider.
	final HashMap<ComponentName, PackageParser.Provider> mProvidersByComponent =
		new HashMap<ComponentName, PackageParser.Provider>();

	......

	private PackageParser.Package scanPackageLI(PackageParser.Package pkg,
			int parseFlags, int scanMode, long currentTime) {
		......

		synchronized (mPackages) {
			......

			// Add the new setting to mPackages
			mPackages.put(pkg.applicationInfo.packageName, pkg);

			......

			int N = pkg.providers.size();
			int i;
			for (i=0; i<N; i++) {
				PackageParser.Provider p = pkg.providers.get(i);
				p.info.processName = fixProcessName(pkg.applicationInfo.processName,
					p.info.processName, pkg.applicationInfo.uid);
				mProvidersByComponent.put(new ComponentName(p.info.packageName,
					p.info.name), p);

				......
			}

			N = pkg.services.size();
			for (i=0; i<N; i++) {
				PackageParser.Service s = pkg.services.get(i);
				s.info.processName = fixProcessName(pkg.applicationInfo.processName,
					s.info.processName, pkg.applicationInfo.uid);
				mServices.addService(s);

				......
			}

			N = pkg.receivers.size();
			r = null;
			for (i=0; i<N; i++) {
				PackageParser.Activity a = pkg.receivers.get(i);
				a.info.processName = fixProcessName(pkg.applicationInfo.processName,
					a.info.processName, pkg.applicationInfo.uid);
				mReceivers.addActivity(a, "receiver");
				
				......
			}

			N = pkg.activities.size();
			for (i=0; i<N; i++) {
				PackageParser.Activity a = pkg.activities.get(i);
				a.info.processName = fixProcessName(pkg.applicationInfo.processName,
					a.info.processName, pkg.applicationInfo.uid);
				mActivities.addActivity(a, "activity");
				
				......
			}

			......
		}

		......

		return pkg;
	}

	......
}
```
到这里整个应用的安装过程就介绍完了。其实整个过程还是很明确，清晰的。
接下来再来总结一下整个启动过程：
> Zygote—>启动SystemServer—>启动ServerThread—>启动PackageManagerService—>扫描特定目录下的apk文件，进行加载—>解析APK的manifest文件，将配置信息加载到`PackageManagerService`中


