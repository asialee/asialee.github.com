---
layout: post
title: Android启动过程详解（4）——SystemServer
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
上一篇博客介绍了ZygoteService的启动过程，在Zygote的启动后首先就会启动SystemServer，所以SystemServer可以看做是Zygote嫡长子。能够成为嫡长子的必然有着非常重要的作用，的确是这样，Android应用框架中的各种Service，例如ActivityManagerService，PacakgeManagerService，WindowManagerService等等重要的系统服务都是在SystemServer中启动的。这篇博客我们就来看看SystemServer的启动过程。

### 1.SystemServer启动
SystemServer是通过Zygote.startSystemServer来启动的，所以首先来看看对应的代码,其对应的实现在dalvik_system_Zygote.c中，这是一个native函数：
```
[-->dalvik_system_Zygote.c]
static voidDalvik_dalvik_system_Zygote_forkSystemServer(
                        const u4* args, JValue* pResult)
{
     pid_tpid;
    //根据参数，fork一个子进程
    pid =forkAndSpecializeCommon(args);

    if (pid > 0) {
       int status;
       gDvm.systemServerPid = pid;//保存system_server的进程id
      //函数退出前须先检查刚创建的子进程是否退出了。
        if(waitpid(pid, &status, WNOHANG) == pid) {
           //如果system_server退出了，Zygote直接干掉了自己
           //看来Zygote和SS的关系异常紧密，简直是生死与共！
            kill(getpid(), SIGKILL);
        }
    }
   RETURN_INT(pid);
}
```
通过上面的代码可以看出，Zygote会fork出一个进程作为SystemServer的宿主，具体的操作是在`forkAndSpecializeCommon`中完成。在完成创建SystemServer的进程之后，Zygote会检查其对应的进程是否退出，如果退出了，Zygote也会杀死自己。
下面，再看看forkAndSpecializeCommon，代码如下所示：
```
[-->dalvik_system_Zygote.c]
static pid_t forkAndSpecializeCommon(const u4*args)
{
    pid_tpid;
    uid_tuid = (uid_t) args[0];
    gid_tgid = (gid_t) args[1];
   ArrayObject* gids = (ArrayObject *)args[2];
    u4debugFlags = args[3];
   ArrayObject *rlimits = (ArrayObject *)args[4];

   //设置信号处理，待会儿要看看这个函数。  
   setSignalHandler();     
    pid =fork(); //fork子进程
   if (pid== 0) {
     //对子进程要根据传入的参数做一些处理，例如设置进程名，设置各种id（用户id，组id等）
   }
......
}
```

最后看看setSignalHandler函数，它由Zygote在fork子进程前调用，代码如下所示：
```
[-->dalvik_system_Zygote.c]
static void setSignalHandler()
{
    interr;
    structsigaction sa;
   memset(&sa, 0, sizeof(sa));
   sa.sa_handler = sigchldHandler;
    err =sigaction (SIGCHLD, &sa, NULL);//设置信号处理函数，该信号是子进程死亡的信号
}
```
直接看这个信号处理函数sigchldHandler
```
static void sigchldHandler(int s)
{
    pid_tpid;
    intstatus;

    while((pid = waitpid(-1, &status, WNOHANG)) > 0) {
             } else if (WIFSIGNALED(status)) {
          }
        }
        //如果死去的子进程是SS，则Zygote把自己也干掉了，这样就做到了生死与共！
        if(pid == gDvm.systemServerPid) {
           kill(getpid(), SIGKILL);
        }
   }
```

### 2.SystemServer的作用
接下来看看SystemServer在启动的过程中具体做了些什么。
```
pid =Zygote.forkSystemServer();
     if(pid == 0) { //子进程处理逻辑：
           handleSystemServerProcess(parsedArgs);
    }
```
从代码中可以看到，当SystemServer被fork出来之后，会调用handleSystemServerProcess来处理自身的启动逻辑。
```
[-->ZygoteInit.java]
private static void handleSystemServerProcess(
       ZygoteConnection.ArgumentsparsedArgs)
      throws ZygoteInit.MethodAndArgsCaller {
      //关闭从Zygote那里继承下来的Socket。 
      closeServerSocket();
      //设置SS进程的一些参数。
      setCapabilities(parsedArgs.permittedCapabilities,
                   parsedArgs.effectiveCapabilities);
        //调用ZygoteInit函数。
      RuntimeInit.zygoteInit(parsedArgs.remainingArgs);
  }
```
 好了，SS走到RuntimeInit了，它的代码在RuntimeInit.java中，如下所示：   
```
[-->RuntimeInit.java]
public static final void zygoteInit(String[]argv)
           throws ZygoteInit.MethodAndArgsCaller {
     //做一些常规初始化
     commonInit();
     //①native层的初始化。
    zygoteInitNative();
     intcurArg = 0;
     for (/* curArg */ ; curArg < argv.length; curArg++) {
           String arg = argv[curArg];
           if (arg.equals("--")) {
               curArg++;
               break;
           } else if (!arg.startsWith("--")) {
               break;
           } else if (arg.startsWith("--nice-name=")) {
               String niceName = arg.substring(arg.indexOf('=') + 1);
               //设置进程名为niceName，也就是"system_server"
               Process.setArgV0(niceName);
           }
        }
       //startClass名为"com.android.server.SystemServer"
       String startClass = argv[curArg++];
       String[] startArgs = new String[argv.length - curArg];
       System.arraycopy(argv, curArg, startArgs, 0, startArgs.length);
       //②调用startClass，也就是com.android.server.SystemServer类的main函数。
       invokeStaticMain(startClass, startArgs);
}
```
上面的代码有两个地方比较重要：
- zygoteInitNative，执行native层的初始化
- invokeStaticMain(startClass, startArgs)：执行SystemServer的main函数。
  接下来分别讲一下以上两点：

####  2.1 `zygoteInitNative`
该函数最终会调用AppMain的onZygoteInit函数：
```
[-->App_main.cpp]
   virtual void onZygoteInit()
    {
        //下面这些东西和Binder有关系
       sp<ProcessState> proc = ProcessState::self();
        if(proc->supportsProcesses()) {
            proc->startThreadPool();//启动一个线程，用于Binder通信。
       }      
}
```

#### 2.2  invokeStaticMain
```
[-->RuntimeInit.java]
private static void invokeStaticMain(StringclassName, String[] argv)
           throws ZygoteInit.MethodAndArgsCaller {
      ......
      //className为"com.android.server.SystemServer"
       Class<?> cl;
       
       try {
           cl = Class.forName(className);
        }catch (ClassNotFoundException ex) {
           throw new RuntimeException(
                    "Missing class wheninvoking static main " + className,
                    ex);
        }

       Method m;
       try {
           //找到com.android.server.SystemServer类的main函数，肯定有地方要调用它
           m = cl.getMethod("main", new Class[] { String[].class });
        }catch (NoSuchMethodException ex) {
           ......
        }catch (SecurityException ex) {
           ......
        }

       int modifiers = m.getModifiers();
       if(! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
           ......
        }
        //抛出一个异常
       throw new ZygoteInit.MethodAndArgsCaller(m, argv);
    }
```
这个函数比较奇怪的一点是，在加载到`com.android.server.SystemServer`类以及找到其对应的main函数之后，并没有直接启动该函数，而是抛出了一个异常，这个异常会在ZygoteInit的main函数中catch，然后才启动对应的面函数。这样做的原因是为了清空堆栈，让用户看到堆栈的栈顶是ZygoteInit.main。

接下来再来看看Zygote在启动后都干了些什么，首先来看看main函数：
```
[-->SystemServer.java]
public static void main(String[] args) {
   ......
   //加载libandroid_servers.so
   System.loadLibrary("android_servers");
   //调用native的init1函数。
   init1(args);
}
```
所以启动过程转移到了init1函数中：
init1是native函数，在com_android_server_SystemServer.cpp中实现：
```
[-->com_android_server_SystemServer.cpp]
extern "C" int system_init();
static voidandroid_server_SystemServer_init1(JNIEnv* env, jobject clazz)
{
   system_init();//调用另外一个函数。
}
```

system_init的实现在system_init.cpp中，它的代码如下所示：
```
[-->system_init.cpp]
extern "C" status_t system_init()
{
   //下面这些调用和Binder有关，我们会在第6章中讲述，这里先不必管它。
   sp<ProcessState> proc(ProcessState::self());
   sp<IServiceManager> sm = defaultServiceManager();
   sp<GrimReaper>grim = new GrimReaper();
   sm->asBinder()->linkToDeath(grim, grim.get(), 0);
   charpropBuf[PROPERTY_VALUE_MAX];
   property_get("system_init.startsurfaceflinger", propBuf,"1");

    if(strcmp(propBuf, "1") == 0) {
        //SurfaceFlinger服务在system_server进程创建
       SurfaceFlinger::instantiate();
    }
     ......

    //调用com.android.server.SystemServer类的init2函数
   AndroidRuntime* runtime =   AndroidRuntime::getRuntime();
   runtime->callStatic("com/android/server/SystemServer","init2");

//下面这几个函数调用和Binder通信有关。

    if (proc->supportsProcesses()) {
        ProcessState::self()->startThreadPool();
       //调用joinThreadPool后，当前线程也加入到Binder通信的大潮中
       IPCThreadState::self()->joinThreadPool();
       }
    returnNO_ERROR;
}
```

init1函数创建了一些系统服务，然后把调用线程加入Binder通信中。不过其间还通过JNI调用了com.android.server.SystemServer类的init2函数，下面就来看看这个init2函数。init2在Java层，代码在SystemServer.java中:
```
[-->SystemServer.java]
public static final void init2() {
   Threadthr = new ServerThread();
   thr.setName("android.server.ServerThread");
   thr.start();//启动一个ServerThread
}
```
启动了一个ServerThread线程。请直接看它的run函数。这个函数比较长，大概看看它干了什么即可。
```
[-->SystemServer.java::ServerThread的run函数]
public void run(){
             ....
  //启动Entropy Service
  ServiceManager.addService("entropy",new EntropyService());
  //启动电源管理服务
  power =new PowerManagerService();
 ServiceManager.addService(Context.POWER_SERVICE, power);
  //启动电池管理服务。
  battery= new BatteryService(context);
 ServiceManager.addService("battery", battery);

   //初始化看门狗
   Watchdog.getInstance().init(context,battery, power, alarm,
                               ActivityManagerService.self());

  //启动WindowManager服务
  wm =WindowManagerService.main(context, power,
                    factoryTest !=SystemServer.FACTORY_TEST_LOW_LEVEL);
 ServiceManager.addService(Context.WINDOW_SERVICE,wm);         

  //启动ActivityManager服务
(ActivityManagerService)ServiceManager.getService("activity")).setWindowManager(wm);
  ......//系统各种重要服务都在这里启动

   Looper.loop();  //进行消息循环，然后处理消息。关于这部分内容参见第5章。
}
```
init2函数比较简单，就是单独创建一个线程，用以启动系统各项服务。至此，SystemServer启动完毕。