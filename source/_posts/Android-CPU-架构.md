---
title: Android CPU 架构
date: 2019-11-12 18:29:44
tags: [CPU 架构]
---

### 0x0001 什么是 ABI

ABI(Application Binary Interface、程序二进制接口)，描述了应用程序和操作系统之间，一个应用和它的库之间，或者应用的组成部分之间的低接口。

由于不同的手机使用不同的 CPU，而不同的 CPU 支持不同的指令集，不同指令集下的数据格式、操作规范不尽相同，所以CPU 与指令集的每种组合都有专属的应用二进制接口，即 ABI，所以在开发过程中，如果我们引入 so 库，就需要为每个 CPU 架构指定
<!-- more -->
相应的 ABI 下的 so 库。


### 0x0002 配置 so 库规则

> 你应该尽可能的提供专为每个ABI优化过的.so文件，但要么全部支持，要么都不支持：你不应该混合着使用。你应该为每个ABI目录提供对应的.so文件。
 
以 arm64-v8a 为例展示上面规则，`arm64-v8a` 是可以向下兼容的，但前提是你的项目里面没有 `arm64-v8a` 的文件夹，这是什么意思呢？以下为例：

* armeabi: a.so、b.so

* arm64-v8a: a.so


在ABI 类型为 arm64-v8a 的手机中使用到 b.so 时，发现存在 arm64-v8a 目录，但是此时该文件夹下没有 b.so，就会产生运行时异常。可以将 `arm64-v8a` 文件夹删除，那么手机在检索 so 库时发现没有 `arm64-v8a` 文件，就会去查找 armeabi 文件夹下是否有需要的 b.so。


### 0x0003 Android开发中常见的指令集体系


|指令集　|　　　　　　　　  厂商|　　　　位数 
|-|-|-|
|x86(x86)　|　　　　　　　　　  Intel　|　　　32|
|x86_64(Intel 64)　　|　　　　　Intel　|　　　64|
|arm64-v8a(ARMV8-A)　|　　　   AMR|　　　　64|
|armeabi(ARM v5)　　　|　　　  ARM　|　　　32|
|armeabi-v7a(ARM v7)　|　　　 ARM　|　　　32|
|mips　　　　　　　　|　　　　 MIPS　|　　　32|
|mips64　　　　　　|　　　　　MIPS　|　　　64|


```
Intel 64 指令集在 x86基础上扩展的
armabi 是针对旧的或者普通的ARM v5 CPU
armabi-v7a 是针对 ARM v7 CPU
arm64-v8a 是针对最新的 ARM v8a CPU的。

特别注意：x86指令集有两种CPU位，既有32位的，也有64位的。
```

```
armeabi 设备只兼容 armeabi；
armeabi-v7a 设备兼容 armeabi-v7a、armeabi；
arm64-v8a 设备兼容 arm64-v8a、armeabi-v7a、armeabi；
X86 设备兼容 X86、armeabi；
X86_64 设备兼容 X86_64、X86、armeabi；
mips64 设备兼容 mips64、mips；
mips 只兼容 mips；
```



很多设备都支持多于一种的ABI。例如 ARM64 和 x86 设备也可以同时运行armeabi-v7a 和 armeabi 的二进制包。但最好是针对特定平台提供相应平台的二进制包，这种情况下运行时就少了一个模拟层（例如x86设备上模拟arm的虚拟层），从而得到更好的性能（归功于最近的架构更新，例如硬件fpu，更多的寄存器，更好的向量化等)，但同时也增大了 APK 的体积，需自行权衡。



### 0x0004 安装时自动解压原生代码


安装文件时，软件包管理器服务将扫描 APK，并查找合适的共享库文件。找到所需库时，软件包管理器会将它们复制到应用的 data 目录 (`data/data/<package_name>/lib/`) 下的 /lib/lib<name>.so。

如果根本没有共享对象文件，应用也会编译并安装，但在运行时会崩溃。

---

**知识链接：**

[Google 官方文档：ABI 管理](https://developer.android.google.cn/ndk/guides/abis)

[Android jniLibs下目录详解 (.so文件)](http://www.jianshu.com/p/b758e36ae9b5)

[Android SO文件的概念、兼容、适配和可能的错误](http://www.jianshu.com/p/cb15ba69fa89)

[关于Android的.so文件你所需要知道的](http://www.jianshu.com/p/cb05698a1968)

[Android开发------关于.so文件的那些事](http://www.jianshu.com/p/7b9ab71a491e)

[Android 关于arm64-v8a、armeabi-v7a、armeabi、x86下的so文件兼容问题](http://www.cnblogs.com/janehlp/p/7473240.html)