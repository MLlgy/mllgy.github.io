---
title: Android系统启动-zygote进程概述
tags:
---

## Android 系统概述

Android 系统中存在两个截然不同的两个世界：

* Java 世界： Google 针对这个世界提供 SDK，在这个世界运行的程序为基于 Dalvik 虚拟机的 Java 程序。
* Native 世界：使用 Native 语言（C或者 C++）开发的程序。

在这样的 Android 世界中存在两个比较重要的进程：Zygote 进程 和 system_server 进程：

* Zygote 进程：和 Android 系统中的 Java 世界有重要的关系。
* system_server 进程：系统重要的服务（Service）都驻留在 Java 世界中,比如 AMS、PMS 都存在该进程中。

这两个进程是在 Android 系统是 Java 世界的半边天，任何一个进程死亡，都会导致系统崩溃。


## Native世界中的 zygote 

zygote 本身是一个 native 的应用程序，与内核、驱动无关，在 init 进程其中过程中，解析 init.rc 文件创建。

zygote 对应的文件为 App_main.cpp，通过源码可以看到重要工作是由 AppRuntime 的 start 函数完成。

在 AppRuntime::start 方法中，涉及关键的点如下：
* 创建虚拟机-startVM
* 注册JNI函数-startReg
* 通过 JNI 调用 Java 函数，具体为调用 ZygoteInit#main 方法


基于以上3点，**它们开创了 Android 系统的 Java 世界，至此进入了 Android 系统中的 Java 世界**。


## Java 世界中 Zygote

`ZygoteInit#main` 涉及的主要流程：


1. 建立 IPC 通信服务端——`registerZygoteSocket`

在 Android 系统中主要通过 Binder 进行 IPC 通信，但是 zygoet 与系统中其他程序没有使用 Binder，而是使用了 Socket，而 registerZygoteSocket 函数就是为了建立这个 Socket。


![](http://gityuan.com/images/android-arch/android-booting.jpg)

2. 预加载类和资源——`preloadClasses`、`preloadResources`

    * 预加载通用类(`/system/etc/preloaded-classes`)
    * drawable和color资源(`com.android.internal.R.array.preloaded_drawables`、`com.android.internal.R.array.preloaded_color_state_lists`、`com.android.internal.R.array.preloaded_freeform_multi_window_drawables`)
    * openGL:`EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY)`
    * 共享库:`System.loadLibrary("android");
        System.loadLibrary("compiler_rt");
        System.loadLibrary("jnigraphics");`

    preloaded-classes 包含的类有很多，所以预加载类执行的时间较长，这是 Android 系统启动比较慢的原因之一。preResources 加载 framework-res.apk 中的资源。


3. 启动 system_serve——`startSystemServer`


这样函数会创建在 Java 世界中 **系统 Service** 所 **驻留的进程 system_server**，如果该进程死了，会导致 zygote 自杀或者重启。

在该函数中 zygote 进程会执行 fork 操作——`Zygote.forkSystemServer`，产生一个 `sysytem_server` 进程。


4. 有求必应之等待请求——`runSelectLoopMode`


在关键点一中，`registerZygoteSocket` 主注册了一个用于 IPC 的 Socket，此 Socket 在runSelectLoopMode发挥作用。


runSelectLoopMode 的具体工作：

1. 处理客户连接和客户请求，客户在 ZygoteServer 中用 ZygoteConnection 表示。
2. 客户的请求由 ZygoteConnection 的 processOneCommand 处理。



## SystemServer 进程概述


SystemServer 进程的名字为 system_server，为 Zygote 进程 fork 产生的 **第一个子进程**。在关于 Zygote 的描述的第三点中，通过 Zygote.forkSystemServer fork Zygote 进程，
而在后续的处理过程中，将 SystemServier 进程的等级提升的特别高，如果 SystemServier 进程被杀死，那么 Zygote 进程也会将自己杀死，做到了 SystemServier 与 Zygote 的同生共死。

## zygote 如何完成分裂


在上面中可以看到，通过调用 Zygote.forkSystemServer 函数最终 fork 出一个 system_server 进程，来 **运行 Android 系统中的重要的 Service（比如 AMS、PMS等）**，接下来就通过 runSelectLoopMode 来处理客户的消息，
那么此处的客户都有谁呢？zygote 又是如何根据客户的请求执行分裂和繁殖呢？

常见的客户有 Activity、Service 等，此处以 Activity 的启动为例，分析 zygote 的无性繁殖。

若启动的 Activity 所在的进程尚未启动，那么就会首先创建进程，然后才会启动相应的 Activity。在以下过程中需要注意的是：AMS 是允许在 system_server 进程中的服务，所以 zygote 的分裂是有 system_server 控制的。


1. AMS 发送请求

```
AMS#startProcessLocked——>
ProcessList#startProcessLocked——>
Process#startProcess——>
Process#start——>
ZygoteProcess#start——>
ZygoteProcess#startViaZygote(在该方法中配置要启动进程的参数、调用了 openZygoteSocketIfNeeded 函数)——>
ZygoteProcess#zygoteSendArgsAndGetResult——>
attemptZygoteSendArgsAndGetResult
```

其中调用了 openZygoteSocketIfNeeded 的函数，**实现了 systemz_server 进程与 Zygote 进程中的 Socket 连接**，并在 attemptZygoteSendArgsAndGetResult 函数中
向 zygote 发送请求，请求的参数中有一个字符串，具体值为“android.app.ActivityThread”。

这一流程也符合 Gityuan 在 Android 系统启动中给出的流程图：

![Android 系统启动流程](http://gityuan.com/images/android-arch/android-booting.jpg)

由于 AMS 运行在 SystemServer 中，所以 SystemServer进程通过 Socket 向 Zygote 进程发送了消息。

2. zygote 进程响应请求，fork 产生新的进程


在上面的关键点阐述中，runSelectLoopMode 会处理由客户端发起的请求，那么具体是如何处理这个来自客户的请求呢？

* ZygoteInit#main

```
public static void main(String argv[]) {
    zygoteServer.runSelectLoop(abiList);
}
```
* runSelectLoop
```
 Runnable runSelectLoop(String abiList) {
     while(true){
         ...
         // 调用 acceptCommandPeer 接收客户端发起的 socket 请求，获得客户端的 Socket 对象，并且构建 ZygoteConnection 对象，对以后的流程很重要
        ZygoteConnection newPeer = acceptCommandPeer(abiList);
        peer.add(newPeer)
         ...
        ZygoteConnection connection = peers.get(pollIndex);
        //处理客户端来的 Socket 请求
        final Runnable command = connection.processOneCommand(this);
        ...
     }
 }
```


* ZygoteConnection#processOneComman

```
 Runnable processOneCommand(ZygoteServer zygoteServer) {
    String[] args;
    //读取客户端 Socket 中的而参数
    args = Zygote.readArgumentList(mSocketReader);
    ZygoteArguments parsedArgs = new ZygoteArguments(args);
    ...
    // fork 产生新的进程
    pid = Zygote.forkAndSpecialize(parsedArgs.mUid, parsedArgs.mGid, parsedArgs.mGids,
                parsedArgs.mRuntimeFlags, rlimits, parsedArgs.mMountExternal, parsedArgs.mSeInfo,
                parsedArgs.mNiceName, fdsToClose, fdsToIgnore, parsedArgs.mStartChildZygote,
                parsedArgs.mInstructionSet, parsedArgs.mAppDataDir, parsedArgs.mTargetSdkVersion);
    // pid = 0,以下在新的进程中执行
    if (pid == 0) {
                zygoteServer.setForkChild();

                zygoteServer.closeServerSocket();
                IoUtils.closeQuietly(serverPipeFd);
                serverPipeFd = null;
                // zygote 分裂成功，产生新的进程，handleChildProc 进行下面的扫尾工作
                return handleChildProc(parsedArgs, childPipeFd, parsedArgs.mStartChildZygote);
    }
 }

```

1. 新进程的操作

在新的进程中执行 `handleChildProc`，注意，**接下来的操作都发生在新进程中。**

* handleChildProc

```
 private Runnable handleChildProc(ZygoteArguments parsedArgs,
            FileDescriptor pipeFd, boolean isZygote) {
        closeSocket();
        if (parsedArgs.mNiceName != null) {
            Process.setArgV0(parsedArgs.mNiceName);
        }
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        if (parsedArgs.mInvokeWith != null) {
            WrapperInit.execApplication(parsedArgs.mInvokeWith,
                    parsedArgs.mNiceName, parsedArgs.mTargetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(),
                    pipeFd, parsedArgs.mRemainingArgs);

        return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                        parsedArgs.mDisabledCompatChanges,
                        parsedArgs.mRemainingArgs, null /* classLoader */);
    }
```
* ZygoteInit#zygoteInit
```
public static final Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
            String[] argv, ClassLoader classLoader) {
        RuntimeInit.redirectLogStreams();
        RuntimeInit.commonInit();
        // 最终会调用 AppRuntime 的 onZygoteInit 函数，建立和 Binder 的联系
        ZygoteInit.nativeZygoteInit();
        return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,
                classLoader);
}
```
* RuntimeInit#applicationInit
```
   protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges,
            String[] argv, ClassLoader classLoader) {
        nativeSetExitWithoutCleanup(true);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
        VMRuntime.getRuntime().setDisabledCompatChanges(disabledCompatChanges);
        final Arguments args = new Arguments(argv);

        // 持有参数，最终调用相应的方法
        return findStaticMain(args.startClass, args.startArgs, classLoader);
    }
```
* RuntimeInit#findStaticMain
```
   protected static Runnable findStaticMain(String className, String[] argv,
            ClassLoader classLoader) {
        Class<?> cl;
        cl = Class.forName(className, true, classLoader);
        Method m;
        m = cl.getMethod("main", new Class[] { String[].class });
        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
        }
        return new MethodAndArgsCaller(m, argv);
    }
```

构建了 MethodAndArgsCaller 对象并返回，最后 `zygoteServer.runSelectLoop` 返回了  MethodAndArgsCaller 对象，

* ZygoteInit#main
```
public static void main(String argv[]) {
    caller = zygoteServer.runSelectLoop(abiList);
    if(caller != null){
        caller.run();
    }
}
```
* MethodAndArgsCaller#run
```
public void run() {
    // 通过反射调用 ActivityThread#main
    mMethod.invoke(null, new Object[] { mArgs });
}
```

* ActivityThread#main
  
```
   public static void main(String[] args) {
        AndroidOs.install();
        CloseGuard.setEnabled(false);
        Environment.initForCurrentUser();
        Process.setArgV0("<pre-initialized>");
        // 创建主线程 Looper
        Looper.prepareMainLooper();
        ...
        ActivityThread thread = new ActivityThread();
        // attach 系统进程
        thread.attach(false, startSeq);
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
        Looper.loop();
    }
```
至此完成了从 Zygote fork 产生新进程，并且在 **新进程** 中执行 `ActivityThread#main` 方法，正式开启新进程的工作。



----

**知识链接**

深入理解 Android(卷1)

[理解Android进程创建流程](http://gityuan.com/2016/03/26/app-process-create/4)