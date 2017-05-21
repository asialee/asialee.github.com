---
layout: post
title: Android启动过程详解（3）——Zygote
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
由于Android系统是基于Linux的，所以在Android系统存在两个不一样的空间，Android空间（Java空间）以及Native空间。系统启动的时候当然是Native空间，所以必须有一个进程来打开Android空间。同时，尽管Android的设计者在淡化进程的概念，强化`Activity`,`Service`,	`BroadcastReceiver`等等组件的概念，但从Linux的视角来看，每一个应用都是寄生在一个进程上的，那么创建进程也同样需要从Native空间去创建。在Android世界中Zygote就担任了这个角色，所以所有应用程序进程的父进程都是Zygote。Zygote的意思是受精卵，所以从名字上就能看出来它的作用。今天就来讨论一下Zygote的作用。

### 1.Zygote的启动
在上一篇博客介绍init进程启动的时候讲到了init.rc文件的解析，其中就有关于Zygote的启动：
```
#zygote service
service zygote /system/bin/app_process -Xzygote/system/bin –zygote \
 --start-system-server 
   socketzygote stream 666  #socket关键字表示OPTION
   onrestart write /sys/android_power/request_state wake #onrestart也是OPTION
   onrestart write /sys/power/state on
   onrestart restart media
```

从中可以看到zygote的启动是通过/system/bin/app_process进程启动的。所以zygote最初的名字叫“app_process”，这个名字是在Android.mk文件中被指定的，但app_process在运行过程中，通过Linux下的pctrl系统调用将自己的名字换成了“zygote”，所以我们通过ps命令看到的进程名是“zygote”。接下来看看app_process的启动：
```
[-->App_main.cpp]
int main(int argc, const char* const argv[])
{
  /*
   Zygote进程由init通过fork而来，我们回顾一下init.rc中设置的启动参数：
    -Xzygote/system/bin --zygote --start-system-server
  */
    mArgC= argc;
    mArgV= argv;
   mArgLen = 0;
    for(int i=0; i<argc; i++) {
       mArgLen += strlen(argv[i]) + 1;
    }
   mArgLen--;
   AppRuntime runtime;
    // 调用Appruntime的addVmArguments，这个函数很简单，读者可以自行分析
    int i= runtime.addVmArguments(argc, argv);
    if (i< argc) {
       //设置runtime的mParentDir为/system/bin
       runtime.mParentDir = argv[i++];
    }
    if (i< argc) {
       arg = argv[i++];
        if(0 == strcmp("--zygote", arg)) {
          //我们传入的参数满足if的条件，而且下面的startSystemServer的值为true
           bool startSystemServer = (i < argc) ?
                    strcmp(argv[i],"--start-system-server") == 0 : false;
           setArgv0(argv0, "zygote");
//设置本进程名为zygote，这正是前文所讲的“换名把戏”。
            set_process_name("zygote");
                 //①调用runtime的start，注意第二个参数startSystemServer为true
           runtime.start("com.android.internal.os.ZygoteInit",
                              startSystemServer);
        }
        ......
    }
         ......
}
```
在这个函数中一共做了两件重要的事：
- 创建Android运行时环境，AppRuntime
- 启动SystemServer
  SystemServer是管理Android所有服务的部件，这一部分放到下一篇博客讲，重点来讲一下AppRumtime的创建。
  ![这里写图片描述](http://img.blog.csdn.net/20161114195525489)
  从上面这幅图可以看到，AppRuntime继承自AndroidRumtime，所以在app_process进程中调用的runtime.start其实是调用的AndroidRumtime的start函数：
```
void AndroidRuntime::start(const char*className, const bool startSystemServer)
{
    //className的值是"com.android.internal.os.ZygoteInit"
    //startSystemServer的值是true
    char*slashClassName = NULL;
    char*cp;
   JNIEnv* env;
    blockSigpipe();//处理SIGPIPE信号
     ......
    constchar* rootDir = getenv("ANDROID_ROOT");
    
if (rootDir == NULL) {
//如果环境变量中没有ANDROID_ROOT,则新增该变量，并设置值为“/system"
       rootDir = “/system";
        ......
       setenv("ANDROID_ROOT", rootDir, 1);
    }
    //① 创建虚拟机
    if(startVm(&mJavaVM, &env) != 0)
        goto bail;
     //②注册JNI函数
    if(startReg(env) < 0) {
        goto bail;
    }
    
   jclass stringClass;
   jobjectArray strArray;
   jstring classNameStr;
   jstring startSystemServerStr;
   stringClass = env->FindClass("java/lang/String");
    //创建一个有两个元素的String数组，即Java代码 String strArray[] = new String[2]
   strArray = env->NewObjectArray(2, stringClass, NULL);
  classNameStr = env->NewStringUTF(className);
  
   //设置第一个元素为"com.android.internal.os.ZygoteInit"
   env->SetObjectArrayElement(strArray, 0, classNameStr);
   startSystemServerStr = env->NewStringUTF(startSystemServer ?
                                                "true" : "false");
   //设置第二个元素为"true"，注意这两个元素都是String类型，即字符串。
   env->SetObjectArrayElement(strArray, 1, startSystemServerStr);
   jclass startClass;
   jmethodID startMeth;
   slashClassName = strdup(className);
   /*
     将字符串“com.android.internal.os.ZygoteInit”中的“. ”换成“/”，
     这样就变成了“com/android/internal/os/ZygoteInit”,这个名字符合JNI规范，
     我们可将其简称为ZygoteInit类。
   */
    for(cp = slashClassName; *cp != '\0'; cp++)
        if(*cp == '.')
           *cp = '/';
   startClass = env->FindClass(slashClassName);
    ......
    //找到ZygoteInit类的static main函数的jMethodId。
   startMeth = env->GetStaticMethodID(startClass, "main",
                                             "([Ljava/lang/String;)V");
     ......
     /*
        ③通过JNI调用Java函数，注意调用的函数是main，所属的类是
          com.android.internal.os.ZygoteInit，传递的参数是
          “com.android.internal.os.ZygoteInit true”，
          调用ZygoteInit的main函数后，Zygote便进入了Java世界！
          也就是说，Zygote是开创Android系统中Java世界的盘古。
     */
      env->CallStaticVoidMethod(startClass,startMeth, strArray);
    //Zygote退出，在正常情况下，Zygote不需要退出。
    if(mJavaVM->DetachCurrentThread() != JNI_OK)
       LOGW("Warning: unable to detach main thread\n");
    if(mJavaVM->DestroyJavaVM() != 0)
       LOGW("Warning: VM did not shut down cleanly\n");
bail:
   free(slashClassName);
}
```
上面这段代码主要做了三件事：
- `startVM`:注册Java虚拟机，并设置虚拟机相关参数
- `startReg`:注册JNI函数，因为后续Java世界用到的一些函数是采用native方式来实现的，所以才必须提前注册这些函数。
- 调用ZygoteInit.main正式进入Java世界

接下来重点讲一下第三点。首先来看看源码：
```
[-->ZygoteInit.java]

public static void main(String argv[]) {
  try {
       SamplingProfilerIntegration.start();
       //①注册Zygote用的socket
       registerZygoteSocket();
       //②预加载类和资源
       preloadClasses();
       preloadResources();
       ......
       // 强制一次垃圾收集
       gc();
      //我们传入的参数满足if分支
      if (argv[1].equals("true")) {
          startSystemServer();//③启动system_server进程
       }else if (!argv[1].equals("false")) {
          thrownew RuntimeException(argv[0] + USAGE_STRING);
        }
      // ZYGOTE_FORK_MODE被定义为false，所以满足else的条件
       if(ZYGOTE_FORK_MODE) {
            runForkMode();
       }else {
          runSelectLoopMode();//④zygote调用这个函数
       }
       closeServerSocket();//关闭socket
        }catch (MethodAndArgsCaller caller) {
           caller.run();//⑤很重要的caller run函数，以后分析
        }catch (RuntimeException ex) {
          closeServerSocket();
           throw ex;
        }
     ......
    }
```
在这个函数中主要做了四件事：
- 创建IPC通信Socket
- 预加载类和资源
- 启动SystemServer
- 监听其他进程发出来的请求

#### 1.1 创建IPC通信接口Socket——registerZygoteSocket
Zygote和其他进程之间的IPC并不是通过binder机制，而是通过一个localsocket来完成的，所以要首先创建一个socket：
```
[-->ZygoteInit.java]
private static void registerZygoteSocket() {
    if(sServerSocket == null) {
        intfileDesc;
        try{
           //从环境变量中获取Socket的fd，还记得第3章init中介绍的zygote是如何启动的吗？
//这个环境变量由execv传入。
          String env = System.getenv(ANDROID_SOCKET_ENV);
          fileDesc = Integer.parseInt(env);
       }
       
       try{
         //创建服务端Socket，这个Socket将listen并accept Client
         sServerSocket= new LocalServerSocket(createFileDescriptor(fileDesc));
       }
       }

}
```

#### 1.2 预加载类和资源
这部分就不细讲了，逻辑比较简单，就是提前加载好一些运行时经常会用到的类和资源，提升系统运行的效率，但这同时会导致系统在启动的时候时间过长，所以这是一个取舍的问题。

#### 1.3 启动SystemServer——startSystemServer
这是Zygote非常重要的一个功能，因为Android启动的所有Service（ActivityManagerService，WindowManagerService，PackageManagerService）都是通过SystemServer来启动的。
```
private static boolean startSystemServer()
           throws MethodAndArgsCaller, RuntimeException {
        //设置参数
       String args[] = {
            "--setuid=1000",//uid和gid等设置
           "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,
                            3001,3002,3003",
           "--capabilities=130104352,130104352",
           "--runtime-init",
           "--nice-name=system_server", //进程名，叫system_server
           "com.android.server.SystemServer", //启动的类名
        };

       ZygoteConnection.Arguments parsedArgs = null;
       int pid;
       try {
          //把上面字符串数组参数转换成Arguments对象。具体内容请读者自行分析。
           parsedArgs = new ZygoteConnection.Arguments(args);
           int debugFlags = parsedArgs.debugFlags;
          //fork一个子进程，看来，这个子进程就是system_server进程。
           pid = Zygote.forkSystemServer(
                    parsedArgs.uid,parsedArgs.gid,
                    parsedArgs.gids,debugFlags, null);
        }catch (IllegalArgumentException ex) {
           throw new RuntimeException(ex);
        }

        if(pid == 0) {
         //① system_server进程的工作
           handleSystemServerProcess(parsedArgs);
        }

       //zygote返回true
       return true;
    }
```
逻辑还是比较简单的，就是通过fork创建出SystemServer进程，具体SystemServer启动如何工作下一篇博客会详细介绍。

#### 1.4 监听其他进程发出来的请求——runSelectLoopMode
registerZygoteSocket中注册了一个用于IPC的Socket，不过那时还没有地方用到它。它的用途将在这个runSelectLoopMode中体现出来，请看下面的代码：
```
[-->ZygoteInit.java]

private static void runSelectLoopMode()
throws MethodAndArgsCaller {
       ArrayList<FileDescriptor> fds = new ArrayList();
       ArrayList<ZygoteConnection> peers = new ArrayList();
       FileDescriptor[] fdArray = new FileDescriptor[4];

      //sServerSocket是我们先前在registerZygoteSocket建立的Socket
       fds.add(sServerSocket.getFileDescriptor());
       peers.add(null);
       int loopCount = GC_LOOP_COUNT;

       while (true) {
           int index;

             try {
               fdArray = fds.toArray(fdArray);
        /*
          selectReadable内部调用select，使用多路复用I/O模型。
          当有客户端连接或有数据时，则selectReadable就会返回。
        */
              index = selectReadable(fdArray);
           }

          else if (index == 0) {
             //如有一个客户端连接上，请注意客户端在Zygote的代表是ZygoteConnection
               ZygoteConnection newPeer = acceptCommandPeer();
               peers.add(newPeer);
               fds.add(newPeer.getFileDesciptor());
           } else {
               boolean done;
              //客户端发送了请求，peers.get返回的是ZygoteConnection
             //后续处理将交给ZygoteConnection的runOnce函数完成。
               done = peers.get(index).runOnce();
        }
    }
```
runSelectLoopMode比较简单，就是：

- 处理客户连接和客户请求。其中客户在Zygote中用ZygoteConnection对象来表示;
- 客户的请求由ZygoteConnection的runOnce来处理。

### 2.处理请求
Zygote在启动过程中的最后一步是进入runSelectLoopMode来监听socket，接收请求。比如当ActivityManagerService在接收到一个打开Activity的请求，并发现该Activity所属的应用并没有启动时就会创建一个进程，具体的过程可以参考我的另一篇博客：[Android应用框架之应用启动过程](http://blog.csdn.net/asialiyazhou/article/details/53049894)


