---
title: Android系统启动-zygote进程篇
tags:
---


* Zygote 进程是 init 进程解析 init.rc 文件创建的。
* 对应可执行程序：app_process，对应的源文件：App_main.cpp


```
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
```



1. 解析启动时传入的参数
2. AndroidRuntime.cpp 
     * 创建虚拟机、注册 JNI 方法
3. 在 AndroidRuntime.start 中通过反射执行 ZygoteInit.main(Java 方法)
    * 为 Zygote 注册 socket
    * 预加载类和资源
    * fork 子进程，用于运行 system_server
    * 启动 system_server



---

1. 解析init.zygote.rc中的参数，创建AppRuntime并调用AppRuntime.start()方法；
2. 调用AndroidRuntime的startVM()方法创建虚拟机，再调用startReg()注册JNI函数；
3. 通过JNI方式调用ZygoteInit.main()，第一次进入Java世界；
4. registerZygoteSocket()建立socket通道，zygote作为通信的服务端，用于响应客户端请求；
5. preload()预加载通用类(类路径 `/system/etc/preloaded-classes`)、drawable和color资源、openGL以及共享库以及WebView，用于提高app启动效率；
    * 预加载通用类(`/system/etc/preloaded-classes`)
    * drawable和color资源(`com.android.internal.R.array.preloaded_drawables`、`com.android.internal.R.array.preloaded_color_state_lists`、`com.android.internal.R.array.preloaded_freeform_multi_window_drawables`)
    * openGL:`EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY)`
    * 共享库:`System.loadLibrary("android");
        System.loadLibrary("compiler_rt");
        System.loadLibrary("jnigraphics");`
6. zygote完毕大部分工作，接下来再通过startSystemServer()，fork得力帮手system_server进程，也是上层framework的运行载体。
7. **zygote功成身退，调用runSelectLoop()，随时待命，当接收到请求创建新进程请求时立即唤醒并执行相应工作**。



至此可以看到 Zygote 的主要作用是用来 fork 新进程，用来运行 App 。