---
title: Android 中的命令行工具
tags:
---
[命令行工具](https://developer.android.google.cn/studio/command-line?hl=zh_cn)



### Android SDK 工具
所在位置：`android_sdk/tools/bin/`

Android SDK 工具是 Android SDK 的一个组件。它包含一整套适用于 Android 的开发和调试工具，并且内置在 Android Studio 中。

### Android SDK 编译工具


所在位置：`android_sdk/build-tools/version/`


此软件包用于编译 Android 应用。这里的工具大多数都是由编译工具调用的，而不是供应用开发者使用的。

如果您使用的是 Android Plugin for Gradle 3.0.0 或更高版本，那么您的项目会自动使用该插件指定的默认版本的 Build Tools。要使用其他版本的 Build Tools，请在模块的 build.gradle 中使用 buildToolsVersion 进行指定:

```
    android {
        buildToolsVersion "29.0.0"
        ...
    }
    
```


### Android SDK 平台工具

所在位置：android_sdk/platform-tools/

Android SDK 平台工具是 Android SDK 的一个组件。 它包含与 Android 平台进行交互的工具，例如 adb、fastboot 和 systrace。开发 Android 应用时需要使用这些工具。如果您想要解锁设备的引导加载程序并为其刷入新的系统映像，则同样需要使用这些工具。


### Android 模拟器

所在位置：`android_sdk/emulator/`

使用 Android 模拟器时需要使用此软件包，主要包含以下情况：

* emulator
  
一种基于 QEMU 的设备模拟工具，可用于在实际的 Android 运行时环境中调试和测试应用。

* mksdcard

可帮助您创建可与模拟器一起使用的磁盘映像，以模拟存在外部存储卡（例如 SD 卡）的情形。