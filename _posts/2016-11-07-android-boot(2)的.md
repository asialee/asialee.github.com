---
layout: post
title: Android启动过程详解（2）——init进程启动逻辑
keywords: 算法
categories: [Android]
tag: [Android]
icon: code
---
init进程是Android系统用户空间中的第一个进程，其进程号也是1，足见其重要性。所以它的责任也是重大的，概括地来说init进程主要做了以下几件事：

> 1. 作为守护进程
> 2. 解析和执行init.rc文件
> 3. 属性服务
> 4. 生成设备驱动节点

接下来文章就着init进程的源码，来一个个分析init进程的工作。首先来看看init进程的源码：
```
int main(int argc, char **argv)  
{  
    int fd_count = 0;  
    struct pollfd ufds[4];  
    char *tmpdev;  
    char* debuggable;  
    char tmp[32];  
    int property_set_fd_init = 0;  
    int signal_fd_init = 0;  
    int keychord_fd_init = 0;  
    bool is_charger = false;  
  
  
    //启动ueventd  
    if (!strcmp(basename(argv[0]), "ueventd"))  
        return ueventd_main(argc, argv);  
  
  
    //启动watchdogd  
    if (!strcmp(basename(argv[0]), "watchdogd"))  
        return watchdogd_main(argc, argv);  
  
  
    /* clear the umask */  
    umask(0);  
  
  
        /* Get the basic filesystem setup we need put 
         * together in the initramdisk on / and then we'll 
         * let the rc file figure out the rest. 
         */  
    //创建并挂在启动所需的文件目录  
    mkdir("/dev", 0755);  
    mkdir("/proc", 0755);  
    mkdir("/sys", 0755);  
  
  
    mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");  
    mkdir("/dev/pts", 0755);  
    mkdir("/dev/socket", 0755);  
    mount("devpts", "/dev/pts", "devpts", 0, NULL);  
    mount("proc", "/proc", "proc", 0, NULL);  
    mount("sysfs", "/sys", "sysfs", 0, NULL);  
  
  
        /* indicate that booting is in progress to background fw loaders, etc */  
    close(open("/dev/.booting", O_WRONLY | O_CREAT, 0000));//检测/dev/.booting文件是否可读写和创建  
  
  
        /* We must have some place other than / to create the 
         * device nodes for kmsg and null, otherwise we won't 
         * be able to remount / read-only later on. 
         * Now that tmpfs is mounted on /dev, we can actually 
         * talk to the outside world. 
         */  
    open_devnull_stdio();//重定向标准输入/输出/错误输出到/dev/_null_  
    klog_init();//log初始化  
    property_init();//属性服务初始化  
  
  
    //从/proc/cpuinfo中读取Hardware名，在后面的mix_hwrng_into_linux_rng_action函数中会将hardware的值设置给属性ro.hardware  
    get_hardware_name(hardware, &revision);  
  
  
    //导入并设置内核变量  
    process_kernel_cmdline();  
  
  
    //selinux相关，暂不分析  
    union selinux_callback cb;  
    cb.func_log = klog_write;  
    selinux_set_callback(SELINUX_CB_LOG, cb);  
  
  
    cb.func_audit = audit_callback;  
    selinux_set_callback(SELINUX_CB_AUDIT, cb);  
  
  
    selinux_initialize();  
    /* These directories were necessarily created before initial policy load 
     * and therefore need their security context restored to the proper value. 
     * This must happen before /dev is populated by ueventd. 
     */  
    restorecon("/dev");  
    restorecon("/dev/socket");  
    restorecon("/dev/__properties__");  
    restorecon_recursive("/sys");  
  
  
    is_charger = !strcmp(bootmode, "charger");//关机充电相关，暂不做分析  
  
  
    INFO("property init\n");  
    if (!is_charger)  
        property_load_boot_defaults();  
  
  
    INFO("reading config file\n");  
    init_parse_config_file("/init.rc");//解析init.rc配置文件  
  
  
    /* 
     * 解析完init.rc后会得到一系列的action等，下面的代码将执行处于early-init阶段的action。 
     * init将action按照执行时间段的不同分为early-init、init、early-boot、boot。 
     * 进行这样的划分是由于有些动作之间具有依赖关系，某些动作只有在其他动作完成后才能执行，所以就有了先后的区别。 
     * 具体哪些动作属于哪个阶段是在init.rc中的配置决定的 
     */  
    action_for_each_trigger("early-init", action_add_queue_tail);  
  
  
    queue_builtin_action(wait_for_coldboot_done_action, "wait_for_coldboot_done");  
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");  
    queue_builtin_action(keychord_init_action, "keychord_init");  
    queue_builtin_action(console_init_action, "console_init");  
  
  
    /* execute all the boot actions to get us started */  
    action_for_each_trigger("init", action_add_queue_tail);  
  
  
    /* skip mounting filesystems in charger mode */  
    if (!is_charger) {  
        action_for_each_trigger("early-fs", action_add_queue_tail);  
        action_for_each_trigger("fs", action_add_queue_tail);  
        action_for_each_trigger("post-fs", action_add_queue_tail);  
        action_for_each_trigger("post-fs-data", action_add_queue_tail);  
    }  
  
  
    /* Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random 
     * wasn't ready immediately after wait_for_coldboot_done 
     */  
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");  
  
  
    queue_builtin_action(property_service_init_action, "property_service_init");  
    queue_builtin_action(signal_init_action, "signal_init");  
    queue_builtin_action(check_startup_action, "check_startup");  
  
  
    if (is_charger) {  
        action_for_each_trigger("charger", action_add_queue_tail);  
    } else {  
        action_for_each_trigger("early-boot", action_add_queue_tail);  
        action_for_each_trigger("boot", action_add_queue_tail);  
    }  
  
  
        /* run all property triggers based on current state of the properties */  
    queue_builtin_action(queue_property_triggers_action, "queue_property_triggers");  
  
  
  
  
#if BOOTCHART  
    queue_builtin_action(bootchart_init_action, "bootchart_init");  
#endif  
  
  
    for(;;) {//init进入无限循环  
        int nr, i, timeout = -1;  
        //检查action_queue列表是否为空。如果不为空则移除并执行列表头中的action  
        execute_one_command();  
        restart_processes();//重启已经死去的进程  
  
  
        if (!property_set_fd_init && get_property_set_fd() > 0) {  
            ufds[fd_count].fd = get_property_set_fd();  
            ufds[fd_count].events = POLLIN;  
            ufds[fd_count].revents = 0;  
            fd_count++;  
            property_set_fd_init = 1;  
        }  
        if (!signal_fd_init && get_signal_fd() > 0) {  
            ufds[fd_count].fd = get_signal_fd();  
            ufds[fd_count].events = POLLIN;  
            ufds[fd_count].revents = 0;  
            fd_count++;  
            signal_fd_init = 1;  
        }  
        if (!keychord_fd_init && get_keychord_fd() > 0) {  
            ufds[fd_count].fd = get_keychord_fd();  
            ufds[fd_count].events = POLLIN;  
            ufds[fd_count].revents = 0;  
            fd_count++;  
            keychord_fd_init = 1;  
        }  
  
  
        if (process_needs_restart) {  
            timeout = (process_needs_restart - gettime()) * 1000;  
            if (timeout < 0)  
                timeout = 0;  
        }  
  
  
        if (!action_queue_empty() || cur_action)  
            timeout = 0;  
  
  
#if BOOTCHART  
        if (bootchart_count > 0) {  
            if (timeout < 0 || timeout > BOOTCHART_POLLING_MS)  
                timeout = BOOTCHART_POLLING_MS;  
            if (bootchart_step() < 0 || --bootchart_count == 0) {  
                bootchart_finish();  
                bootchart_count = 0;  
            }  
        }  
#endif  
        //等待事件发生  
        nr = poll(ufds, fd_count, timeout);  
        if (nr <= 0)  
            continue;  
  
  
        for (i = 0; i < fd_count; i++) {  
            if (ufds[i].revents == POLLIN) {  
                if (ufds[i].fd == get_property_set_fd())//处理属性服务事件  
                    handle_property_set_fd();  
                else if (ufds[i].fd == get_keychord_fd())//处理keychord事件  
                    handle_keychord();  
                else if (ufds[i].fd == get_signal_fd())//处理  
                    handle_signal();//处理SIGCHLD信号  
            }  
        }  
    }  
  
  
    return 0;  
}
```

### 1.文件挂载，生成驱动节点

首先来看看main函数第一块比较重要的部分，文件挂载：
```
 /* Get the basic filesystem setup we need put 
         * together in the initramdisk on / and then we'll 
         * let the rc file figure out the rest. 
         */  
    //创建并挂在启动所需的文件目录  
    mkdir("/dev", 0755);  
    mkdir("/proc", 0755);  
    mkdir("/sys", 0755);  


    mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");  
    mkdir("/dev/pts", 0755);  
    mkdir("/dev/socket", 0755);  
    mount("devpts", "/dev/pts", "devpts", 0, NULL);  
    mount("proc", "/proc", "proc", 0, NULL);  
    mount("sysfs", "/sys", "sysfs", 0, NULL);  


        /* indicate that booting is in progress to background fw loaders, etc */  
    close(open("/dev/.booting", O_WRONLY | O_CREAT, 0000));//检测/dev/.booting文件是否可读写和创建  


        /* We must have some place other than / to create the 
         * device nodes for kmsg and null, otherwise we won't 
         * be able to remount / read-only later on. 
         * Now that tmpfs is mounted on /dev, we can actually 
         * talk to the outside world. 
         */  
    open_devnull_stdio();//重定向标准输入/输出/错误输出到/dev/_null_  
    klog_init();//log初始化 
```

这部分代码还是比较简单的，就是挂载一些文件的挂载。值得说明的是：
> 1.tmpfs是一种虚拟内存文件系统，通常情况下该文件系统是常驻ram的，所以其读取速度要高于内存和磁盘。而/dev目录则存放了访问硬件设备所需的驱动程序文件，将tmpfs作用于驱动目录最主要的原因还是提高硬件设备的访问速度。
> 2. devpts是一种虚拟终端文件系统。
> 3. proc是一个基于内存的文件系统，其主要的作用是完成内核与应用之间的数据交换。
> 4. sysfs是一种特殊的文件系统，在Linux 2.6中引入，用于将系统中的设备组织成层次结构，并向用户模式程序提供详细的内核数据结构信息，将proc、devpts、devfs三种文件系统统一起来。
> 5. /dev /proc /sysfs 等等目录都是系统运行是目录，在Android系统编译时是不存在的，它们都是由init进程创建的。当系统终止时他们也会消失。

最后构造的目录如下：
![这里写图片描述](http://img.blog.csdn.net/20161110004933402)

另外，open_devnull_stdio()函数的作用是重定向标准输入/输出/错误输出到/dev/_null_。klog_init()用于初始化log，通过其实现可以看出log被打印到/dev/__kmsg__文件中。主要在代码中最后通过fcntl和unlink使得/dev/__kmsg__不可被访问，这就保证了只有log程序才可以访问。

### 2. 解析和执行init.rc文件
init.rc文件定义了在系统启动时需要执行的动作也需要启动的服务，这些服务对于整个Android系统的正常运行都是至关重要的，其中大家熟知的ZygoteService就是在init.rc文件中指定需要执行的。所以首先来看看init.rc文件的语法。

#### 2.1 init.rc文件介绍
init.rc文件并不是普通的配置文件，而是由一种被称为“Android初始化语言”（Android Init Language，这里简称为AIL）的脚本写成的文件。首先来看一下init.rc文件
```
#笔者注：引入其他rc文件，这部分文件也将被执行
import /init.environ.rc
import /init.usb.rc
import /init.${ro.hardware}.rc
import /init.trace.rc

on early-init
	# Set init and its forked children's oom_adj
	write /proc/1/oom_adj -16

on init 
	#动作列表
	mkdir /system
	mkdir /data 0771 system system
	mkdir /cache 0770 system cache
	mkdir /config 0500 root root

on boot
	...

on charger
	class_start charger

on property:vold.decrypt=trigger_reset_main
	class_reset main

service evened /sbin/ueventd
service health /sbin/healthd

...
#zygote service
service zygote /system/bin/app_process -Xzygote/system/bin –zygote \

 --start-system-server 

    socketzygote stream 666  #socket关键字表示OPTION

   onrestart write /sys/android_power/request_state wake #onrestart也是OPTION

   onrestart write /sys/power/state on

   onrestart restart media
```

以上就是init.rc文件的节选。下面先来介绍一下其对应的语法。AIL由如下4部分组成。
​	
- 动作（Actions）
- 命令（Commands）
- 服务（Services）
- 选项（Options）

 AIL在编写时需要分成多个部分（Section），而每一部分的开头需要指定Actions或Services。也就是说，每一个Actions或Services确定一个Section。而所有的Commands和Options只能属于最近定义的Section。如果Commands和Options在第一个Section之前被定义，它们将被忽略。
Actions和Services的名称必须唯一。如果有两个或多个Action或Service拥有同样的名称，那么init在执行它们时将抛出错误，并忽略这些Action和Service。
Actions的语法格式如下：
> on trigger
> command
> command
> command

所以action是以on开头的，init.rc中定义的early-init，init，boot，early-boot都是action。comman的就是这个action需要执行的命令。这些指令很好理解，都是Linux下的指令。以init为例，在这个阶段创建了几个文件，并且设置了对应的权限和user。
```
on init 
	#动作列表
	mkdir /system
	mkdir /data 0771 system system
	mkdir /cache 0770 system cache
	mkdir /config 0500 root root
```
![这里写图片描述](http://img.blog.csdn.net/20161112163030002)

接下来再来看看service的结构。
> service [name] [pathname] [ argument ]*  
      [option]  
      [option]

比如servicemanager
> service servicemanager /system/bin/servicemanager  
    class core  
    user system  
    group system  
    critical  
    onrestart restart zygote  
    onrestart restart media  
    onrestart restart surfaceflinger  
    onrestart restart drm

Services的选项是服务的修饰符，可以影响服务如何以及怎样运行。服务支持的选项如下：

1.  critical
    表明这是一个非常重要的服务。如果该服务4分钟内退出大于4次，系统将会重启并进入 Recovery （恢复）模式。
2.  disabled
    表明这个服务不会同与他同trigger （触发器）下的服务自动启动。该服务必须被明确的按名启动。
3.  setenv [name] [value]
    在进程启动时将环境变量[name]设置为[value]。
4.  socket [name][type] [perm] [ user [ group ] ]
    创建一个unix域的名为/dev/socket/name 的套接字，并传递它的文件描述符给已启动的进程。type 必须是 "dgram","stream" 或"seqpacket"。用户和组默认是0。
5.  user username
    在启动这个服务前改变该服务的用户名。此时默认为 root。
6.  group groupname [groupname ]*
    在启动这个服务前改变该服务的组名。除了（必需的）第一个组名，附加的组名通常被用于设置进程的补充组（通过setgroups函数），档案默认是root。
7.  oneshot
    服务退出时不重启。
8.  class name
    指定一个服务类。所有同一类的服务可以同时启动和停止。如果不通过class选项指定一个类，则默认为"default"类服务。
9.  onrestart
    当服务重启，执行一个命令

#### 2.2 init.rc文件解析
接下来正式进入到init.rc文件的解析过程，在init程序中有这样一行代码：
```
init_parse_config_file("/init.rc");//解析init.rc配置文件
```
其对应的实现是
```
int init_parse_config_file(const char *fn)  
{  
    char *data;  
    data = read_file(fn, 0);  
    if (!data) return -1;  
  
    parse_config(fn, data);  
    DUMP();  
    return 0;  
}
```
接下来再来看看parse_config
```
static void parse_config(const char *fn, char *s)  
{  
    struct parse_state state;  
    struct listnode import_list;//导入链表，用于保持在init.rc中通过import导入的其他rc文件  
    struct listnode *node;  
    char *args[INIT_PARSER_MAXARGS];  
    int nargs;  
  
  
    nargs = 0;  
    state.filename = fn;//初始化filename的值为init.rc文件  
    state.line = 0;//初始化行号为0  
    state.ptr = s;//初始化ptr指向s，即read_file读入到内存中的init.rc文件的首地址  
    state.nexttoken = 0;//初始化nexttoken的值为0  
    state.parse_line = parse_line_no_op;//初始化行解析函数  
  
  
    list_init(&import_list);  
    state.priv = &import_list;  
  
  
    for (;;) {  
        switch (next_token(&state)) {  
        case T_EOF://如果返回为T_EOF，表示init.rc已经解析完成，则跳到parser_done解析import进来的其他rc脚本  
            state.parse_line(&state, 0, 0);  
            goto parser_done;  
        case T_NEWLINE:  
            state.line++;//一行读取完成后，行号加1  
            if (nargs) {//如果刚才解析的一行为语法行（非注释等），则nargs的值不为0，需要对这一行进行语法解析  
                int kw = lookup_keyword(args[0]);//init.rc中每一个语法行均是以一个keyword开头的，因此args[0]即表示这一行的keyword  
                if (kw_is(kw, SECTION)) {  
                    state.parse_line(&state, 0, 0);  
                    parse_new_section(&state, kw, nargs, args);  
                } else {  
                    state.parse_line(&state, nargs, args);  
                }  
                nargs = 0;//复位  
            }  
            break;  
        case T_TEXT://将nexttoken解析的一个text保存到args字符串数组中，nargs的最大值为INIT_PARSER_MAXARGS（64），即init.rc中一行最多不能超过INIT_PARSER_MAXARGS个text（单词）  
            if (nargs < INIT_PARSER_MAXARGS) {  
                args[nargs++] = state.text;  
            }  
            break;  
        }  
    }  
  
  
......
}
```
代码的总体逻辑比较简单，解析文件中的关键字、换行符等等解析各个属性，然后添加到对应的section中。后续代码逻辑也比较信息，读者可以自行看源码，这里因为篇幅原因就不再继续介绍源码了。
解析init.rc文件的最终效果就是解析文件中的各个action和service，然后将其添加到action_list和service_list中。供后续执行list中的action和service。

而在解析完成后，有这样一段代码：
```
action_for_each_trigger("early-init", action_add_queue_tail);  
  
  
    queue_builtin_action(wait_for_coldboot_done_action, "wait_for_coldboot_done");  
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");  
    queue_builtin_action(keychord_init_action, "keychord_init");  
    queue_builtin_action(console_init_action, "console_init");  
  
  
    /* execute all the boot actions to get us started */  
    action_for_each_trigger("init", action_add_queue_tail);  
  
  
    /* skip mounting filesystems in charger mode */  
    if (!is_charger) {  
        action_for_each_trigger("early-fs", action_add_queue_tail);  
        action_for_each_trigger("fs", action_add_queue_tail);  
        action_for_each_trigger("post-fs", action_add_queue_tail);  
        action_for_each_trigger("post-fs-data", action_add_queue_tail);  
    }  
  
  
    /* Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random 
     * wasn't ready immediately after wait_for_coldboot_done 
     */  
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");  
  
  
    queue_builtin_action(property_service_init_action, "property_service_init");  
    queue_builtin_action(signal_init_action, "signal_init");  
    queue_builtin_action(check_startup_action, "check_startup");  
  
  
    if (is_charger) {  
        action_for_each_trigger("charger", action_add_queue_tail);  
    } else {  
        action_for_each_trigger("early-boot", action_add_queue_tail);  
        action_for_each_trigger("boot", action_add_queue_tail);  
    }  
  
  
        /* run all property triggers based on current state of the properties */  
    queue_builtin_action(queue_property_triggers_action, "queue_property_triggers");  
  
  
  
  
#if BOOTCHART  
    queue_builtin_action(bootchart_init_action, "bootchart_init");  
#endif  

```
这部分代码就比较好理解了，执行已经解析到的service和action。

#### 2.3 Zygote service的启动
看了以上代码可能大家会奇怪，对应执行的都是action，怎么没有看到service的执行，其实service的启动已经包含在了action的执行当中了。以Zygote为例，这个service的启动就是放在boot过程中的。接下来重点讲一下。
在init.rc下的action中有一个command叫做
`class_start [classservice]`
指令的意思就是启动属于classservice这个类别的所有service。而在boot阶段就有一个command是：
`start_service default`
所以在boot阶段会启动所有的default的service。或许你已经猜到了，没错，Zygote就属于default类型，因而在boot阶段会被执行。关于Zygote如何被设置为default，具体的工作在parse_service阶段完成的，读者可以自行分析源码，这里不做介绍。
相应的启动服务的操作也比较简单,在init进程中通过fork，创建一个新的进程，用于运行Zygote，相关代码如下：
```
if(pid == 0) {
    //pid为零，我们在子进程中
       struct socketinfo *si;
       struct svcenvinfo *ei;
       char tmp[32];
       int fd, sz;
//得到属性存储空间的信息并加到环境变量中，后面在属性服务一节中会碰到使用它的地方。
       get_property_workspace(&fd, &sz);
       add_environment("ANDROID_PROPERTY_WORKSPACE", tmp);
        //添加环境变量信息
       for (ei = svc->envvars; ei; ei = ei->next)
           add_environment(ei->name, ei->value);
        //根据socketinfo创建socket
       for (si = svc->sockets; si; si = si->next) {
           int s = create_socket(si->name,
                                  !strcmp(si->type,"dgram") ?
                                  SOCK_DGRAM :SOCK_STREAM,
                                  si->perm,si->uid, si->gid);
           if (s >= 0) {
               //在环境变量中添加socket信息。
                publish_socket(si->name, s);
           }
        }
       ......//设置uid，gid等
     setpgid(0, getpid());
       if(!dynamic_args) {
        /*
执行/system/bin/app_process，这样就进入到app_process的main函数中了。
fork、execve这两个函数都是Linux系统上常用的系统调用。
        */
            if (execve(svc->args[0], (char**)svc->args, (char**) ENV) < 0) {
              ......
           }
        }else {
          ......
    }
```
为什么要执行/system/bin/app_process?再来看看Zygote在init.rc文件中的定义：
```
#zygote service
service zygote /system/bin/app_process -Xzygote/system/bin –zygote \
 --start-system-server 
    socketzygote stream 666  #socket关键字表示OPTION
   onrestart write /sys/android_power/request_state wake #onrestart也是OPTION
   onrestart write /sys/power/state on
   onrestart restart media
```
是不是一目了然了，启动zygote service就是通过执行/system/bin/app_process进程来实现的。另外在进程中还创建了一个socket，这个socket用于和init进程进行通信，下文会介绍。


### 3守护进程
在init的main函数的最后，init会进入到一个无限循环中，作为一个守护进程。
```
for(;;) {//init进入无限循环  
        int nr, i, timeout = -1;  
        //检查action_queue列表是否为空。如果不为空则移除并执行列表头中的action  
        execute_one_command();  
        restart_processes();//这里会重启所有flag标志为SVC_RESTARTING的service  
  
        if (!property_set_fd_init && get_property_set_fd() > 0) {  
            ufds[fd_count].fd = get_property_set_fd();  
            ufds[fd_count].events = POLLIN;  
            ufds[fd_count].revents = 0;  
            fd_count++;  
            property_set_fd_init = 1;  
        }  
        if (!signal_fd_init && get_signal_fd() > 0) {  
            ufds[fd_count].fd = get_signal_fd();  
            ufds[fd_count].events = POLLIN;  
            ufds[fd_count].revents = 0;  
            fd_count++;  
            signal_fd_init = 1;  
        }  
        if (!keychord_fd_init && get_keychord_fd() > 0) {  
            ufds[fd_count].fd = get_keychord_fd();  
            ufds[fd_count].events = POLLIN;  
            ufds[fd_count].revents = 0;  
            fd_count++;  
            keychord_fd_init = 1;  
        }  
  
        if (process_needs_restart) {  
            timeout = (process_needs_restart - gettime()) * 1000;  
            if (timeout < 0)  
                timeout = 0;  
        }  
  
        if (!action_queue_empty() || cur_action)  
            timeout = 0;  
  
#if BOOTCHART  
        if (bootchart_count > 0) {  
            if (timeout < 0 || timeout > BOOTCHART_POLLING_MS)  
                timeout = BOOTCHART_POLLING_MS;  
            if (bootchart_step() < 0 || --bootchart_count == 0) {  
                bootchart_finish();  
                bootchart_count = 0;  
            }  
        }  
#endif  
        //等待事件发生  
        nr = poll(ufds, fd_count, timeout);  
        if (nr <= 0)  
            continue;  
  
        for (i = 0; i < fd_count; i++) {  
            if (ufds[i].revents == POLLIN) {  
                if (ufds[i].fd == get_property_set_fd())//处理属性服务事件  
                    handle_property_set_fd();  
                else if (ufds[i].fd == get_keychord_fd())//处理keychord事件  
                    handle_keychord();  
                else if (ufds[i].fd == get_signal_fd())//处理  
                    handle_signal();//处理SIGCHLD信号  
            }  
        }  
    }  
```
这里以Zygote的重启为例，介绍一下守护进程的处理机制。当子进程退出时，init的这个信号处理函数会被调用:
```
static void sigchld_handler(int s)
{  
   write(signal_fd, &s, 1); //往signal_fd write数据
}
```
signal_fd，就是在init中通过socketpair创建的两个socket中的一个，既然会往这个signal_fd中发送数据，那么另外一个socket就一定能接收到，这样就会导致init从poll函数中返回
```
nr =poll(ufds, fd_count, timeout);
 ......
 if(ufds[i].revents == POLLIN) {
	 ......
	 else if (ufds[i].fd == get_signal_fd())//处理  
          handle_signal();//处理SIGCHLD信号  
 }
```
接下来就会调用handle_signal()函数来处理。init进程会找到死掉的那个service，然后先杀死该service创建的进程，清理socket信息，设置标示为SVC_RESTARTING,然后执行该service on restart中的COMMAND。接下来在init守护进程的下一次循环中启动：
```
......
restart_processes();//这里会重启所有flag标志为SVC_RESTARTING的service
......
```

### 4.属性服务
在init进程的main()中，我们可以看见如下语句：
```
queue_builtin_action(property_service_init_action, "property_service_init");
```
这句话的意思是启动action链表中的property服务：
```
static int property_service_init_action(int nargs, char **args)  
{  
    /* read any property files on system or data and 
     * fire up the property service.  This must happen 
     * after the ro.foo properties are set above so 
     * that /data/local.prop cannot interfere with them. 
     */  
    start_property_service();  
    return 0;  
}
```
start_property_service()的实现如下：
```
void start_property_service(void)  
{  
    int fd;  
  
    load_properties_from_file(PROP_PATH_SYSTEM_BUILD);  
    load_properties_from_file(PROP_PATH_SYSTEM_DEFAULT);  
    load_override_properties();  
    /* Read persistent properties after all default values have been loaded. */  
    load_persistent_properties();  
  
    fd = create_socket(PROP_SERVICE_NAME, SOCK_STREAM, 0666, 0, 0);  
    if(fd < 0) return;  
    fcntl(fd, F_SETFD, FD_CLOEXEC);  
    fcntl(fd, F_SETFL, O_NONBLOCK);  
  
    listen(fd, 8);  
    property_set_fd = fd;  
}
```
关于这段代码做一些说明：

在Android中定义了5个存储属性的文件，它们分别是：
```
#define PROP_PATH_RAMDISK_DEFAULT  "/default.prop"  
#define PROP_PATH_SYSTEM_BUILD     "/system/build.prop"  
#define PROP_PATH_SYSTEM_DEFAULT   "/system/default.prop"  
#define PROP_PATH_LOCAL_OVERRIDE   "/data/local.prop"  
#define PROP_PATH_FACTORY        "/factory/factory.prop"
```
load_persistent_properties()用来加载persist开头的属性文件，这些属性文件是需要保存到永久介质上的，这些属性文件存储在/data/property目录下。
在start_property_service（）的最后创建了一个名为"property_service"的socket，启动监听，并将socket的句柄保存在property_set_fd中。
处理设置属性请求

接收属性设置请求的地方在init进程中：
```
nr = poll(ufds, fd_count, timeout);  
if (nr <= 0)  
    continue;  
  
for (i = 0; i < fd_count; i++) {  
    if (ufds[i].revents == POLLIN) {  
        if (ufds[i].fd == get_property_set_fd())  
            handle_property_set_fd();  
        else if (ufds[i].fd == get_keychord_fd())  
            handle_keychord();  
        else if (ufds[i].fd == get_signal_fd())  
            handle_signal();  
    }  
}
```
可以看到，在init接收到其他进程设置属性的请求时，会调用handle_property_set_fd()函数进程处理,最后完成属性设置。


参考文章：

- https://my.oschina.net/wizardmerlin/blog/511926
- http://wiki.jikexueyuan.com/project/deep-android-v1/init.html 


