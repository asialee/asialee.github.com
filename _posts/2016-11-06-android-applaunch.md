---
layout: post
title: Android应用框架之应用启动过程
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
> 1. 在Android的应用框架中，`ActivityManagerService`是非常重要的一个组件，尽管名字叫做`ActivityManagerService`，但通过之前的博客介绍，我们知道，四大组件的创建都是有AMS来完成的，其实不仅是应用程序中的组件，连Android应用程序本身也是AMS负责启动的。AMS本身运行在一个独立的进程中，当系统决定要在一个新的进程中启动一个Activity或者Service时就会先启动这个进程。而AMS启动进程的过程是从startProcessLocked启动的。
>
>    ### 1.ActivityManagerService.startProcessLocked
>    ```
>    public final class ActivityManagerService extends ActivityManagerNative  
>            implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
>        ......  
>        private final void startProcessLocked(ProcessRecord app,  
>                    String hostingType, String hostingNameStr) {  
>            ......  
>            try {  
>                int uid = app.info.uid;  
>                int[] gids = null;  
>                try {  
>                    gids = mContext.getPackageManager().getPackageGids(  
>                        app.info.packageName);  
>                } catch (PackageManager.NameNotFoundException e) {  
>                    ......  
>                }  
>                ......  
>                int debugFlags = 0;  
>                ......  
>                int pid = Process.start("android.app.ActivityThread",  
>                    mSimpleProcessManagement ? app.processName : null, uid, uid,  
>                    gids, debugFlags, null);  
>                ......  
>            } catch (RuntimeException e) {  
>                ......  
>            }  
>        }  
>        ......  
>    }  
>    ```
>    可以看到，函数会调用Process.start函数来创建一个进程，其中第一个参数"android.app.ActivityThread"是需要加载的类，而在完成这个类的加载之后就会运行`ActivityThread.main`函数。
>
>    ### 2.Process.start
>    ```
>    public class Process {
>    	......
>    	public static final int start(final String processClass,
>    		final String niceName,
>    		int uid, int gid, int[] gids,
>    		int debugFlags,
>    		String[] zygoteArgs)
>    	{
>    		if (supportsProcesses()) {
>    			try {
>    				return startViaZygote(processClass, niceName, uid, gid, gids,
>    					debugFlags, zygoteArgs);
>    			} catch (ZygoteStartFailedEx ex) {
>    				......
>    			}
>    		} else {
>    			......
>    			return 0;
>    		}
>    	}
>    	......
>    }
>    ```
>    这个函数最后会调用startViaZygote来创建进程，而Zygote正是Android孵化进程的服务，所有的进程都是通过Zygotefork出来的，所以这里创建进程的任务又落到了Zygote头上了。
>
>    ### 3.`Process.startViaZygote`
>    ```
>    public class Process {
>    	......
>    	private static int startViaZygote(final String processClass,
>    			final String niceName,
>    			final int uid, final int gid,
>    			final int[] gids,
>    			int debugFlags,
>    			String[] extraArgs)
>    			throws ZygoteStartFailedEx {
>    		int pid;
>
>    		synchronized(Process.class) {
>    			ArrayList<String> argsForZygote = new ArrayList<String>();
>
>    			// --runtime-init, --setuid=, --setgid=,
>    			// and --setgroups= must go first
>    			argsForZygote.add("--runtime-init");
>    			argsForZygote.add("--setuid=" + uid);
>    			argsForZygote.add("--setgid=" + gid);
>    			if ((debugFlags & Zygote.DEBUG_ENABLE_SAFEMODE) != 0) {
>    				argsForZygote.add("--enable-safemode");
>    			}
>    			if ((debugFlags & Zygote.DEBUG_ENABLE_DEBUGGER) != 0) {
>    				argsForZygote.add("--enable-debugger");
>    			}
>    			if ((debugFlags & Zygote.DEBUG_ENABLE_CHECKJNI) != 0) {
>    				argsForZygote.add("--enable-checkjni");
>    			}
>    			if ((debugFlags & Zygote.DEBUG_ENABLE_ASSERT) != 0) {
>    				argsForZygote.add("--enable-assert");
>    			}
>
>    			//TODO optionally enable debuger
>    			//argsForZygote.add("--enable-debugger");
>
>    			// --setgroups is a comma-separated list
>    			if (gids != null && gids.length > 0) {
>    				StringBuilder sb = new StringBuilder();
>    				sb.append("--setgroups=");
>
>    				int sz = gids.length;
>    				for (int i = 0; i < sz; i++) {
>    					if (i != 0) {
>    						sb.append(',');
>    					}
>    					sb.append(gids[i]);
>    				}
>
>    				argsForZygote.add(sb.toString());
>    			}
>
>    			if (niceName != null) {
>    				argsForZygote.add("--nice-name=" + niceName);
>    			}
>
>    			argsForZygote.add(processClass);
>
>    			if (extraArgs != null) {
>    				for (String arg : extraArgs) {
>    					argsForZygote.add(arg);
>    				}
>    			}
>    			pid = zygoteSendArgsAndGetPid(argsForZygote);
>    		}
>    	}
>    	......
>    }
>    ```
>    函数里面最为重要的工作就是组装argsForZygote参数，这些参数将告诉Zygote具体的启动选项，例如"--runtime-init"就表示要为新启动的运行程序初始化运行库。然后调用zygoteSendAndGetPid函数进一步操作。
>
>    ### 4.`Process.zygoteSendAndGetPid`
>    ```
>    public class Process {
>    	......
>
>    	private static int zygoteSendArgsAndGetPid(ArrayList<String> args)
>    			throws ZygoteStartFailedEx {
>    		int pid;
>
>    		openZygoteSocketIfNeeded();
>
>    		try {
>    			/**
>    			* See com.android.internal.os.ZygoteInit.readArgumentList()
>    			* Presently the wire format to the zygote process is:
>    			* a) a count of arguments (argc, in essence)
>    			* b) a number of newline-separated argument strings equal to count
>    			*
>    			* After the zygote process reads these it will write the pid of
>    			* the child or -1 on failure.
>    			*/
>
>    			sZygoteWriter.write(Integer.toString(args.size()));
>    			sZygoteWriter.newLine();
>
>    			int sz = args.size();
>    			for (int i = 0; i < sz; i++) {
>    				String arg = args.get(i);
>    				if (arg.indexOf('\n') >= 0) {
>    					throw new ZygoteStartFailedEx(
>    						"embedded newlines not allowed");
>    				}
>    				sZygoteWriter.write(arg);
>    				sZygoteWriter.newLine();
>    			}
>
>    			sZygoteWriter.flush();
>
>    			// Should there be a timeout on this?
>    			pid = sZygoteInputStream.readInt();
>
>    			if (pid < 0) {
>    				throw new ZygoteStartFailedEx("fork() failed");
>    			}
>    		} catch (IOException ex) {
>    			......
>    		}
>    		return pid;
>    	}
>    	......
>    }
>    ```
>    这里的sZygoteWriter是一个Socket写入流，是由openZygoteSocketIfNeeded函数打开的。而这个Socket由frameworks/base/core/java/com/android/internal/os/ZygoteInit.java文件中的ZygoteInit类在runSelectLoopMode函数侦听的。这个类会返回一个ZygoteConnection实例，并执行ZygoteConnection的runOnce函数。
>
>    ### 5.`ZygoteConnection.runOnce`
>    ```
>    class ZygoteConnection {
>    	......
>
>    	boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {
>    		String args[];
>    		Arguments parsedArgs = null;
>    		FileDescriptor[] descriptors;
>
>    		try {
>    			args = readArgumentList();
>    			descriptors = mSocket.getAncillaryFileDescriptors();
>    		} catch (IOException ex) {
>    			......
>    			return true;
>    		}
>
>    		......
>
>    		/** the stderr of the most recent request, if avail */
>    		PrintStream newStderr = null;
>
>    		if (descriptors != null && descriptors.length >= 3) {
>    			newStderr = new PrintStream(
>    				new FileOutputStream(descriptors[2]));
>    		}
>
>    		int pid;
>    		
>    		try {
>    			parsedArgs = new Arguments(args);
>
>    			applyUidSecurityPolicy(parsedArgs, peer);
>    			applyDebuggerSecurityPolicy(parsedArgs);
>    			applyRlimitSecurityPolicy(parsedArgs, peer);
>    			applyCapabilitiesSecurityPolicy(parsedArgs, peer);
>
>    			int[][] rlimits = null;
>
>    			if (parsedArgs.rlimits != null) {
>    				rlimits = parsedArgs.rlimits.toArray(intArray2d);
>    			}
>
>    			pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid,
>    				parsedArgs.gids, parsedArgs.debugFlags, rlimits);
>    		} catch (IllegalArgumentException ex) {
>    			......
>    		} catch (ZygoteSecurityException ex) {
>    			......
>    		}
>
>    		if (pid == 0) {
>    			// in child
>    			handleChildProc(parsedArgs, descriptors, newStderr);
>    			// should never happen
>    			return true;
>    		} else { /* pid != 0 */
>    			// in parent...pid of < 0 means failure
>    			return handleParentProc(pid, descriptors, parsedArgs);
>    		}
>    	}
>    	......
>    }
>    ```
>    真正创建进程的代码在`Zygote.forkAndSpecialize`,通过Zygote来fork出一个新的进程作为应用进程。fork函数会有两个返回，其中一个在父进程，一个在子进程，其中自进程的进程号会为0，所以按照上面的代码，这里会执行handleChildProc。
>
>    ### 6.`ZygoteConnection.handleChildProc`
>    ```
>    class ZygoteConnection {
>    	......
>    	private void handleChildProc(Arguments parsedArgs,
>    			FileDescriptor[] descriptors, PrintStream newStderr)
>    			throws ZygoteInit.MethodAndArgsCaller {
>    		......
>    		if (parsedArgs.runtimeInit) {
>    			RuntimeInit.zygoteInit(parsedArgs.remainingArgs);
>    		} else {
>    			......
>    		}
>    	}
>    	......
>    }
>    ```
>    因为在创建的时候传入了“--runtime-init”，所以这里会运行RuntimeInit.zygoteInit。
>    ```
>    public class RuntimeInit {
>    	......
>
>    	public static final void zygoteInit(String[] argv)
>    			throws ZygoteInit.MethodAndArgsCaller {
>    		// TODO: Doing this here works, but it seems kind of arbitrary. Find
>    		// a better place. The goal is to set it up for applications, but not
>    		// tools like am.
>    		System.setOut(new AndroidPrintStream(Log.INFO, "System.out"));
>    		System.setErr(new AndroidPrintStream(Log.WARN, "System.err"));
>
>    		commonInit();
>    		zygoteInitNative();
>
>    		int curArg = 0;
>    		for ( /* curArg */ ; curArg < argv.length; curArg++) {
>    			String arg = argv[curArg];
>
>    			if (arg.equals("--")) {
>    				curArg++;
>    				break;
>    			} else if (!arg.startsWith("--")) {
>    				break;
>    			} else if (arg.startsWith("--nice-name=")) {
>    				String niceName = arg.substring(arg.indexOf('=') + 1);
>    				Process.setArgV0(niceName);
>    			}
>    		}
>
>    		if (curArg == argv.length) {
>    			Slog.e(TAG, "Missing classname argument to RuntimeInit!");
>    			// let the process exit
>    			return;
>    		}
>
>    		// Remaining arguments are passed to the start class's static main
>
>    		String startClass = argv[curArg++];
>    		String[] startArgs = new String[argv.length - curArg];
>
>    		System.arraycopy(argv, curArg, startArgs, 0, startArgs.length);
>    		invokeStaticMain(startClass, startArgs);
>    	}
>    	......
>    }
>    ```
>    这里有两个关键的函数调用，一个是zygoteInitNative函数调用，一个是invokeStaticMain函数调用，前者就是执行Binder驱动程序初始化的相关工作了，正是由于执行了这个工作，才使得进程中的Binder对象能够顺利地进行Binder进程间通信，而后一个函数调用，就是执行进程的入口函数，这里就是执行startClass类的main函数了，而这个startClass即是我们在Step 1中传进来的"android.app.ActivityThread"值，表示要执行android.app.ActivityThread类的main函数。
>
>    ### 7. `Zygote.invokeStaticMain`
>    ```
>    public class ZygoteInit {
>    	......
>
>    	static void invokeStaticMain(ClassLoader loader,
>    			String className, String[] argv)
>    			throws ZygoteInit.MethodAndArgsCaller {
>    		Class<?> cl;
>
>    		try {
>    			cl = loader.loadClass(className);
>    		} catch (ClassNotFoundException ex) {
>    			......
>    		}
>
>    		Method m;
>    		try {
>    			m = cl.getMethod("main", new Class[] { String[].class });
>    		} catch (NoSuchMethodException ex) {
>    			......
>    		} catch (SecurityException ex) {
>    			......
>    		}
>
>    		int modifiers = m.getModifiers();
>    		......
>
>    		/*
>    		* This throw gets caught in ZygoteInit.main(), which responds
>    		* by invoking the exception's run() method. This arrangement
>    		* clears up all the stack frames that were required in setting
>    		* up the process.
>    		*/
>    		throw new ZygoteInit.MethodAndArgsCaller(m, argv);
>    	}
>    	......
>    }
>    ```
>    从代码中可以看到，通过ClassLoader加载对应的android.app.ActivityThread类，然后再获取到对应的main函数句柄，最后调用该类的main函数。不过这里的调用方式比较有意思，不知直接调用，而是通过抛出一个异常。这样做的方式是为了清空堆栈，让系统认为新进程是从ActivityThread的main函数开始的。
>
>    ### 8.ActivityThread.main
>    ```
>    public final class ActivityThread {
>    	......
>
>    	public static final void main(String[] args) {
>    		SamplingProfilerIntegration.start();
>
>    		Process.setArgV0("<pre-initialized>");
>
>    		Looper.prepareMainLooper();
>    		if (sMainThreadHandler == null) {
>    			sMainThreadHandler = new Handler();
>    		}
>
>    		ActivityThread thread = new ActivityThread();
>    		thread.attach(false);
>
>    		if (false) {
>    			Looper.myLooper().setMessageLogging(new
>    				LogPrinter(Log.DEBUG, "ActivityThread"));
>    		}
>    		Looper.loop();
>
>    		if (Process.supportsProcesses()) {
>    			throw new RuntimeException("Main thread loop unexpectedly exited");
>    		}
>
>    		thread.detach();
>    		String name = (thread.mInitialApplication != null)
>    			? thread.mInitialApplication.getPackageName()
>    			: "<unknown>";
>    		Slog.i(TAG, "Main thread of " + name + " is now exiting");
>    	}
>
>    	......
>    }
>    ```
>     从这里我们可以看出，这个函数首先会在进程中创建一个ActivityThread对象,然后进入消息循环中,这样，我们以后就可以在这个进程中启动Activity或者Service了。