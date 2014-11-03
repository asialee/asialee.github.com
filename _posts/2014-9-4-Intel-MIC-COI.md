---
layout: post
title: Intel MIC架构下COI框架介绍
keywords: MIC COI
categories: [HPC]
tag: [HPC,MIC,COI]
icon: code
---

开始介绍之前先写一下历史背景，为了最大限度地提高计算速度，单一地提高一个核的主频以提高计算速率的方法已经不再适用。所以向量机、超标量计算机等纷纷出现，并行计算也再度成为了一个热门的方向。现有的并行计算架构主要有两个：GPGPU（通用GPU）以及Intel的MIC（Many Integrated Core）架构。通用GPU加速主要是利用GPU本身具有多线程的特性，将计算密集型任务迁移到GPU上并将计算任务划分到多个线程内同时进行计算，再将计算结果传回已达到提高计算速率的效果。主要的用到的技术就是CUDA。但是CUDA有一个很大的缺点就是：要编写CUDA的代码工作量很大，通常情况下需要将原有的在CPU下可以运行的代码做大量的改动。而这一点正是Intel MIC架构的优势，通常情况下可以在CPU上直接运行的代码也可以在MIC卡上直接运行。如果想了解更多MIC的相关知识可以持续关注我的博客，我将在接下来对MIC进行更详细的介绍。

MIC编程主要有native模式、对称模式以及offload模式。native模式就是程序只在MIC卡上运行，并没有将CPU的运行效能全部运行起来；对称模式就是CPU和MIC卡运行的程序完全相同，可以看成是对等的节点；而offload模式则是以CPU上运行程序为主，但是将一些计算密集型的任务offload到MIC上进行运算。从代码的角度来看，在MIC上运行的并不是一套完整的代码，而只是一些代码片段。但是使用过offload模式进行大规模数值模式程序编写的人应该都有体会，offload的方式有这么两个缺点：1、数据传输会是一个瓶颈，很多时候不是运算不够快而是数据的传输跟不上；2、offload模式通常只适用于一些较为扁平的数据结构的操作。博主手头的程序都是大量使用了面向对象特性的程序，offload模式使用起来很麻烦，对于程序的结构也不能很好的维护。而针对这些问题，本着提高传输效率以及增强编码人员自由度的目的，Intel推出了COI（Coprocessor Offload Infrastructure）。

#COI基本概念#

COI是MIC架构下的分载模式的一个库，mic虽然提供了通过简单的编译制导语句（#pragma offload target）的方式来将部fen代码和数据分载到Xeon Phi上进行计算。但这样的方式太过简单，且自由度较小。但如果使用COI，用户可以获取更大的自由度，包括控制CPU和MIC的同步，控制MIC卡上程序的创建和退出；起止端的异步操作；起止端的数据缓冲。为开发更加灵活的MIC程序提供了便利。

利用COI编写的程序在编译的时候就会生成两个可执行文件。一个在CPU上执行，一个在MIC上执行。启动程序只需要调用CPU端的可执行文件即可。系统就会在需要的时候把MIC端可执行文件和所需要的库发送到MIC端并执行MIC端的程序。接下来开始详细介绍COI。

术语说明：因为COI可以实现CPU到MIC的分载，同时也可以实现MIC到CPU的分载，所以在下文中使用source和sink来表示起止端。

#基本概念#
###Enumeration：COIEngine，COISysinfo###

列举出硬件的信息：如MIC卡的数量，线程数，核数，cache等等。示例代码如下：

{% highlight C++ %}
COIENGINE               engine;  
COIFUNCTION             func[1];  

const char*             SINK_NAME = "coi_simple_sink_mic";  

// Make sure there is an Intel(r) Xeon Phi(tm) device available  
//                                                                                                                                                                                                                                    
CHECK_RESULT(  
COIEngineGetCount(COI_ISA_MIC, &num_engines));  

printf("%u engines available\n", num_engines);  

// If there isn't at least one engine, there is something wrong   
if (num_engines < 1)  
{  
    printf("ERROR: Need at least 1 engine\n");  
    return -1;  
} 
{% endhighlight %}


###Process Management:COIProcess###

COIProcess是指source端在sink端创建的一个进程在source端创建的一个进程在source端的一个句柄。source端负责该进程的创建，开启以及销毁等工作。其主要的作用有：

- 抽象sink端的进程的运行
- 提供开启和停止远程进程的各种API已经load动态链接库
- 提供在远端查询函数并执行函数的功能

具体示例代码如下：
{%highlight C++ %}
COIRESULT               result = COI_ERROR;  
COIPROCESS              proc;  
COIENGINE               engine;       
result = COIProcessCreateFromFile(  
engine,         // The engine to create the process on.  
SINK_NAME,      // The local path to the sink side binary to launch.  
0, NULL,        // argc and argv for the sink process.  
false, NULL,    // Environment variables to set for the sink process.  
true, NULL,     // Enable the proxy but don't specify a proxy root path.  
0,              // The amount of memory to pre-allocate  
// and register for use with COIBUFFERs.  
NULL,           // Path to search for dependencies  
&proc           // The resulting process handle.  
);  
if (result != COI_SUCCESS)  
{  
    printf("COIProcessCreateFromFile result %s\n",  
    COIResultGetName(result));  
    return -1;  
}  

printf("Sink process created, press enter to destroy it.\n");  
getchar();  

// Destroy the process  
//  
result = COIProcessDestroy(  
proc,           // Process handle to be destroyed  
-1,             // Wait indefinitely until main() (on sink side) returns  
false,          // Don't force to exit. Let it finish executing  
// functions enqueued and exit gracefully  
&sink_return,   // Don't care about the exit result.  
&exit_reason  
);  

if (result != COI_SUCCESS)  
{  
    printf("COIProcessDestroy result %s\n", COIResultGetName(result));  
    return -1;  
}  
{% endhighlight %}

###执行流：COIPipeline###
COIPipeline类似于RPC（Remote Procedure Call）的机制，可以讲一系列指令序列插入到COIPipeline中，这些指令可以顺序地在sink端执行，这里的指令序列主要是要在远程调用的函数序列。其主要有以下几个重要的性质：

- 在COIPipeline中插入的函数会在sink端按序执行。
- COIPipeline就是一种远程调用的机制。因为可以在远端调用完整函数，所以有了比单一offload更多的自由度。
- COIPipeline在插入函数时除了插入该函数需要的参数外，还可以传递一块buffer
- 从source到sink端的数据传输使用的是SCIF

示例代码如下：（source端）

{% highlight C++ %}
CHECK_RESULT(  
COIProcessCreateFromFile(  
engine,             // The engine to create the process on.  
SINK_NAME,          // The local path to the sink side binary to launch.  
0, NULL,            // argc and argv for the sink process.  
false, NULL,        // Environment variables to set for the sink  
// process.  
true, NULL,         // Enable the proxy but don't specify a proxy root  
// path.  
0,                  // The amount of memory to pre-allocate  
// and register for use with COIBUFFERs.  
NULL,               // Path to search for dependencies  
&proc               // The resulting process handle.  
));   
printf("Created sink process %s\n", SINK_NAME);  

// Pipeline:  
// After a sink side process is created, multiple pipelines can be created  
// to that process. Pipelines are queues where functions(represented by  
// COIFUNCTION) to be executed on sink side can be enqueued.  

// The following call creates a pipeline associated with process created  
// earlier.  
CHECK_RESULT(  
COIPipelineCreate(  
proc,           // Process to associate the pipeline with  
NULL,           // Do not set any sink thread affinity for the pipeline  
0,              // Use the default stack size for the pipeline thread  
&pipeline       // Handle to the new pipeline  
));  
printf("Created pipeline\n");  


// Retrieve handle to function belonging to sink side process  

const char* func_name = "Foo";  
CHECK_RESULT(  
COIProcessGetFunctionHandles(  
proc,       // Process to query for the function  
1,          // The number of functions to look up  
&func_name, // The name of the function to look up  
func        // A handle to the function  
));  
printf("Got handle to sink function %s\n", func_name);  

const char *misc_data = "Hello COI";  
int strlength =  (int)strlen(misc_data) + 1;  

// Enough to hold the return value  

char* return_value = (char*) malloc(strlength);  
if (return_value == NULL) {  
fprintf(stderr, "failed to allocate return value\n");  
return -1;  
}  


// Enqueue the function for execution  
// Pass in misc_data and return value pointer to run function  
// Get an event to wait on until the run function completion  
CHECK_RESULT(  
COIPipelineRunFunction(  
pipeline, func[0],         // Pipeline handle and function handle  
0, NULL, NULL,             // Buffers and access flags  
0, NULL,                   // Input dependencies  
misc_data,   strlength,    // Misc Data to pass to the function  
return_value, strlength,   // Return values that will be passed back  
&completion_event          // Event to signal when it completes  
));  
printf("Called sink function %s(\"%s\" [%d bytes])\n",  
func_name, misc_data, strlength);  

// Now wait indefinitely for the function to complete  
CHECK_RESULT(  
COIEventWait(  
1,                         // Number of events to wait for  
&completion_event,         // Event handles  
-1,                        // Wait indefinitely  
true,                      // Wait for all events  
NULL, NULL                 // Number of events signaled  
// and their indices  
));  
printf("Function returned \"%s\"\n", return_value);  
{% endhighlight %}

sink端：

{% highlight C++ %}
// main is automatically called whenever the source creates a process.  
// However, once main exits, the process that was created exits.  
int main(int argc, char** argv)  
{  
    UNUSED_ATTR COIRESULT result;  
    UNREFERENCED_PARAM (argc);  
    UNREFERENCED_PARAM (argv);  

    // Functions enqueued on the sink side will not start executing until  
    // you call COIPipelineStartExecutingRunFunctions(). This call is to  
    // synchronize any initialization required on the sink side  

    result = COIPipelineStartExecutingRunFunctions();  

    assert(result == COI_SUCCESS);  

    // This call will wait until COIProcessDestroy() gets called on the source  
    // side. If COIProcessDestroy is called without force flag set, this call  
    // will make sure all the functions enqueued are executed and does all  
    // clean up required to exit gracefully.  

    COIProcessWaitForShutdown();  

    return 0;  
}  

// Prototype of run function that can be retrieved on the source side.  
// Copies misc data to return pointer.  
COINATIVELIBEXPORT  
void Foo (uint32_t         in_BufferCount,  
void**           in_ppBufferPointers,  
uint64_t*        in_pBufferLengths,  
void*            in_pMiscData,  
uint16_t         in_MiscDataLength,  
void*            in_pReturnValue,  
uint16_t         in_ReturnValueLength)  
{  

    UNREFERENCED_PARAM(in_BufferCount);  
    UNREFERENCED_PARAM(in_ppBufferPointers);  
    UNREFERENCED_PARAM(in_pBufferLengths);  
    UNREFERENCED_PARAM(in_pMiscData);  
    UNREFERENCED_PARAM(in_MiscDataLength);  

    assert (in_MiscDataLength>=in_ReturnValueLength);  
    if(in_pMiscData!=NULL && in_pReturnValue!=NULL)  
    {     
        memcpy(in_pReturnValue, in_pMiscData, in_ReturnValueLength);  
    }  
}  
{% endhighlight %}


source端的COIEventWait暂时可以不关心，接下来会详细介绍。通过上述代码可以发现，在source端，将对应的函数插入COIPipeline队列之后，该函数将会在sink端执行。但前提是该函数必须在sink端的代码中被申明。不过被插入到在COIPipeline中的函数并不是会立即在sink端执行。首先是要在sink端调用COIPipelineStartExecutingRunFunctions（）之后才能被执行。另外如果被插入的执行函数有相关的buffer，那么函数也必须在buffer可用的时候才能执行，此外通过COIEvent也可以来控制函数的执行。更多的内容会在后面讲到。

###COIBuffer###

- COIBuffer用于管理在远程设备上的数据。
- Buffer可以通过传入执行函数给远程端点，也可以直接使用读写API。
- COI的runtime（运行时环境）来管理buffer
- Buffer用于在source和sink端的数据传输，buffer可以使source和sink端的读写异步，隐藏掉通信延迟。
- Buffer可能是位于device也可能是位于host端的物理内存中。
- Buffer实际是利用SCIF的内存窗口实现的
- 数据的传输实际利用的是readfrom/writeto  API
- map操作可以访问到buffer对应的区域，而不需要将其数据移到host上

COIBuffer的操作比较复杂，更多的细节会在后续的博客中涉及到。


###COIEvent###
COIEvent可以用来创建依赖关系，从而使source和sink端进行同步，可以通过创建事件然后等待事件来达到同步，有点类似于MPI中的MPI_Barrier函数。一个函数的执行可以有一个先导COIEvent，只有当该event被消费之后，位于COIPipeline中的对应函数才能被执行（该函数此时必须位于COIPipeline队列的首位）。同时，当该函数被执行结束之后，也可以产生一个事件，远端的程序可以通过等待对应的时候来进行同步操作。具体看示例代码：

source端：
{% highlight C++ %}
// This tutorial demonstrates:  
// 1. Registering a User event  
// 2. Pass the event to the sink side                                                                                                                                                                                                         
// 3. Signaling the event from the sink side and using it  
//    to synchronize on the source side.  

// It first enqueues a run function with a registered user event (which  
// is not signaled) as input dependency. Then a second function is enqueued  
// on a different pipeline that signals the user event.  

// User events are one shot events. Once they are signaled they  
// can't be signaled again. You have to register them again to enable  
// signaling.   


//Create two pipelines  

CHECK_RESULT(  
COIPipelineCreate(  
proc,            // Process to associate the pipeline with  
NULL,            // Do not set any sink thread affinity for the pipeline  
0,               // Use the default stack size for the pipeline thread  
&pipeline[0]     // Handle to the new pipeline  
));  

CHECK_RESULT(  
COIPipelineCreate(  
proc,            // Process to associate the pipeline with  
NULL,            // Do not set any sink thread affinity for the pipeline  
0,               // Use the default stack size for the pipeline thread  
&pipeline[1]     // Handle to the new pipeline  
));  
printf("Created sink process %s and two pipelines\n", SINK_NAME);  

// Retrieve handle to functions belonging to sink side process  

const char* names[] = {"Return2","SignalUserEvent"};  

CHECK_RESULT(  
COIProcessGetFunctionHandles(  
proc,        // Process to query for the function  
2,           // The number of functions to query  
names,       // The name of the function  
func         // A handle to the function  
));  
printf("Got handles to functions %s and %s\n", names[0], names[1]);  


uint64_t return_value = 0;  


COIEVENT  user_event;  

// Register this event so that it can be signaled  
CHECK_RESULT(  
COIEventRegisterUserEvent(&user_event));  
printf("Registered user event\n");  


// Now pass this registered user event as an input dependency to the run  
// function. This run function will not be started until the user event  
// is signaled.  
CHECK_RESULT(  
COIPipelineRunFunction(  
pipeline[0], func[0],                  // Pipeline handle and function  
// handle  
0, NULL, NULL,                         // Buffers and access flags to  
// pass to the function  
1, &user_event,                        // Input dependencies  
NULL, 0,                               // Misc data to pass to  
// the function  
&return_value, sizeof(return_value),   // Return value passed back  
// from the function  
&completion_event                      // Event to signal when  
// the function is complete  
));  
printf("Enqueued sink function %s depending on user event\n", names[0]);  

// Sleep for 2 sec which is enough for run function to be started on sink  
// side  
#ifndef _WIN32  
sleep(2);  
#else  
Sleep(2000);  
#endif  
// Now try waiting for the completion_event. It should return  
// COI_TIME_OUT_REACHED (as the event isn't signaled)  
if(COIEventWait(1, &completion_event, 0, true, NULL, NULL) !=  
COI_TIME_OUT_REACHED)  
{  
printf("Error: Did not execute as expected\n");  
return -1;  
}  
printf("As expected, event wait timed out\n");  

// User event handles can be passed down to run function as misc  
// data (or via buffers) and on sink side can be type-casted back to  
// COIEVENT object to signal them.  
CHECK_RESULT(  
COIPipelineRunFunction(  
pipeline[1], func[1],                   // Pipeline handle and function  
// handle  
0, NULL, NULL,                          // Buffers and access flags to  
// pass to the function  
0, NULL,                                // Input dependencies  
&user_event, sizeof(user_event),        // Misc data to pass to  
// the function  
NULL,0,                                 // Return value passed back  
// from the function  
NULL                                    // Event to signal when  
// the function is complete  
));  
printf("Enqueued sink function %s passing user event as misc arg\n",  
names[1]);  

// Wait until the user event is signaled  
CHECK_RESULT(  
COIEventWait(  
1,                      // Number of events to wait for  
&user_event,          // Event handles  
-1,                     // Wait indefinitely  
true,                   // Wait for all events  
NULL, NULL              // Number of events signaled  
// and their indices  
));  
printf("Successfully waited for user event (signaled sink side)\n");  

// Once a user event is signaled the first run function will be able to  
// proceed. Wait until the function finishes (-1 wait indefinite)  
CHECK_RESULT(  
COIEventWait(  
1,                      // Number of events to wait for  
&completion_event,    // Event handles  
-1,                     // Wait indefinitely  
true,                   // Wait for all events  
NULL, NULL              // Number of events signaled  
// and their indices  
));  
printf("Sink function %s completed since user event signaled\n", names[0]);  


// Unregister the event to cleanup  
CHECK_RESULT(  
COIEventUnregisterUserEvent(user_event));  
{% endhighlight %}

sink端

{% highlight c++ %}
// main is automatically called whenever the source creates a process.  
// However, once main exits, the process that was created exits.  
int main(int argc, char** argv)  
{  
UNUSED_ATTR COIRESULT result;  
UNREFERENCED_PARAM (argc);  
UNREFERENCED_PARAM (argv);  

// Functions enqueued on the sink side will not start executing until  
// you call COIPipelineStartExecutingRunFunctions()  
// This call is to synchronize any initialization required on the sink side  

result = COIPipelineStartExecutingRunFunctions();  

assert(result == COI_SUCCESS);  

// This call will wait until COIProcessDestroy() gets called on the source  
// side. If COIProcessDestroy is called without force flag set, this call  
// will make sure all the functions enqueued are executed and does all  
// clean up required to exit gracefully.  
COIProcessWaitForShutdown();  

return 0;  
}  

// Prototype of run functions that can be retrieved on the sink side  

// This Function just returns 2  
COINATIVELIBEXPORT  
void Return2(uint32_t         in_BufferCount,  
void**           in_ppBufferPointers,  
uint64_t*        in_pBufferLengths,  
void*            in_pMiscData,  
uint16_t         in_MiscDataLength,  
void*            in_pReturnValue,  
uint16_t         in_ReturnValueLength)  
{  
UNREFERENCED_PARAM(in_BufferCount);  
UNREFERENCED_PARAM(in_ppBufferPointers);  
UNREFERENCED_PARAM(in_pBufferLengths);  
UNREFERENCED_PARAM(in_pMiscData);  
UNREFERENCED_PARAM(in_MiscDataLength);  

if (sizeof(uint64_t) <= in_ReturnValueLength)  
{     
*(uint64_t*)(in_pReturnValue) = 2;  
}     
}  

//Assumes a user_event is passed as Misc_data and signals it  
COINATIVELIBEXPORT  
void SignalUserEvent(uint32_t         in_BufferCount,  
void**           in_ppBufferPointers,  
uint64_t*        in_pBufferLengths,  
void*            in_pMiscData,  
uint16_t         in_MiscDataLength,  
void*            in_pReturnValue,  
uint16_t         in_ReturnValueLength)  
{  
UNREFERENCED_PARAM(in_BufferCount);  
UNREFERENCED_PARAM(in_ppBufferPointers);  
UNREFERENCED_PARAM(in_pBufferLengths);  
UNREFERENCED_PARAM(in_MiscDataLength);  
UNREFERENCED_PARAM(in_pReturnValue);  
UNREFERENCED_PARAM(in_ReturnValueLength);  

COIEVENT user_event;  

assert(in_pMiscData != NULL);  
assert(in_MiscDataLength >= sizeof(user_event));  

memcpy(&user_event, in_pMiscData, sizeof(user_event));  

COIEventSignalUserEvent(user_event);  
} 
{% endhighlight %}