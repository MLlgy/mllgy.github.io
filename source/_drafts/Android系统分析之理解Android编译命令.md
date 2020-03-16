---
title: Android系统分析之理解Android编译命令
tags:
---


在自己看到这篇文章时，自己正好在学习 Makefile 的基本语法及相关书写，参见 [跟我一起写Makefile](https://www.androidos.net.cn/codebook/Makefile/Introduce.md)，以及 Android 中 ndk-build 脚本的相关知识，参见[ndk-build](https://developer.android.google.cn/ndk/guides/ndk-build?hl=zh_cn)。在此时能够遇到这篇文章 : [理解Android编译命令](http://gityuan.com/2016/03/19/android-build/)，从多角度学习编译的相关知识，无疑是幸运的。



### 编译 Android 源码命令

使用如下命令可以编译 Android 系统源码：

```
source setenv.sh //初始化编译环境，包括后面的lunch和make指令
lunch //指定此次编译的目标设备以及编译类型
make -j12 //开始编译，默认为编译整个系统，其中-j12代表的是编译的job数量为12
```
利用 `source` 或小数点（.）都可以将配置文件的内容读进来目前的 shell 环境中！

将配置文件写入目前的 shell 中后，可以在 **当前 shell** 中执行配置文件的函数，下例：

test.sh

```
#!/bin/bash
funcation test(){
    echo "hello"
}
```

此时在命令行中键入 test，是可以执行 test.sh 中的 test 函数的。

>$ test

打印日志：

> hello

注意 test 命令只在当前 shell 中有效，如果将当前 shell 关闭，那么 test 将无法执行相应动作，如果想该命令永久有效，可以将 test.sh 写到相应的配置文件中或 PATH 中，比如 .bash_profile 等。

其实编译 Android 源码时使用到的所有命令，比如 m、mm、mma 等，都是 setenv.sh 脚本中的函数，具体可以存在哪些函数可以参见链接。


### 编译系统

Android 编译系统是 Android 源码的一部分，用于编译 Android 系统，Android SDK 以及相关文档。该编译系统是由 Make 文件、Shell 以及 Python 脚本共同组成，其中最为重要的便是 Make 文件。


#### makefile 分类

* 系统核心 Makefile 文件

文件路径位于 /build/core,定义了 Build 系统的框架，其他 makefile 文件都是基于该框架编写的。

* 针对产品的 Makefile 文件

路径位于 /device，定义了具体某个型号手机的 Makefile 文件，一般依据公司名和产品名划分两个目录，比如 /device/tencent/heisa 。

* 针对模块的 Makefile 文件

Android 系统分为多个独立模块，每个模块都有自己的 makefile 文件，统一命名为 Android.mk，该文件定义了当前模块的编译方式。 Build 系统会扫描整个源码树目录中的 Android.mk 文件，执行相应模块的编译工作。

#### 编译指令

正如 test.sh  所展示的那样，将 setenv.sh 加载到当前 shell 环境中，该文件声明了终端可用的命令


#### 编译产物

make 后的产物，位于 /out 目录中，关注以下几个目录：


* /out/host

Android 开发工具的产物，包含 SDK 的各种工具，比如 adb、aapt 等。

* /out/target/common

通用的一些编译产物，包括 Java 应用代码和 Java 库。

* /out/target/product/[product_name]

针对特定设备的编译产物以及平台相关C/C++代码和二进制文件。


同时在 /out/target/product/[product_name] 目录下，有几个我们关注的镜像文件：

* system.img

挂载为根分区，主要包含Android OS的系统文件；
* ramdisk.img

主要包含init.rc文件和配置文件等；

* userdata.img

被挂载在/data，主要包含用户以及应用程序相关的数据；

* boot.img
* reocovery.img

通过 fastboot 刷机时，需要使用到这几个镜像文件。


### Android.mk 文件


在 Android 系统代码中，每一个模块的所有文件通常都有一个自己的而文件夹，在该文件夹下存在名为 Android.mk 的文件。通过 make 进行编译源码正是依据 makefile 文件。


编译系统正是以 **模块为单位** 进行编译，每个模块都有唯一的模块名，一个模块可以有依赖多个其他模块，**模块间的依赖关系就是通过模块名来引用的**。也就是说当模块需要依赖一个jar包或者apk时，必须先将jar包或apk定义为一个模块，然后再依赖相应的模块。


为方便模块编译，编译系统设置了很多的编译环境变量、便捷函数,可以查看
Android NDK ndk-build 中关于 Android.mk 和 Application.mk  的编写的相关文档： [ndk-build](https://developer.android.google.cn/ndk/guides/ndk-build?hl=zh_cn)。


---

**知识链接：**


[跟我一起写Makefile](https://www.androidos.net.cn/codebook/Makefile/Introduce.md)

[理解Android编译命令](http://gityuan.com/2016/03/19/android-build/)

[ndk-build 关于 .mk 文件的编写](https://developer.android.google.cn/ndk/guides/ndk-build?hl=zh_cn)