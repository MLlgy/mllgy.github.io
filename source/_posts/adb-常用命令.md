---
title: adb 常用命令
date: 2019-03-26 11:32:49
tags: []
---

### 概述

adb(Android Debug Briage) 一个通用的命令行工具，允许使用它与模拟器或 Android 设备进行通信。 adb 为一个 客户端-服务器程序，包含三个组件：
1. 客户端。发送命令，在电脑端运行，发出 adb 命令从命令行终端调用客户端。
2. 后台程序。在相应的设备(模拟器、真机)上运行命令，作为`后台进程` 在 `设备`上运行。
3. 服务器。管理客户端和后台程序之间的通信，在 `开发计算机` 上作为 `后台进程` 运行。 每个设备将获取一对按顺序排列的端口，一个用于控制台的偶数号端口和一个用于 adb 连接的奇数号端口。


android_sdk/platform-tools/ 中找到 adb 工具。

### adb 工作方式

1. 启动 客户端，如果没有 adb 服务器进程，则启动服务器进程。服务器启动后，与本地的 TCP 端口 5037 绑定，它监听从 adb 客户端发送的命令(所有的 adb 命令均使用 5037 端口与 adb 服务器通信)。
2. 服务器与所有运行的模拟器或设备连接。它扫描5555 到 5585 之间的奇数号端口查找设备。 服务器一旦发现 adb 后台程序，它将与该端口的设备连接。
3. 当服务器与所有的设备连接后，使用 adb 命令来访问这些设备。


### 通过 WLAN 连接设备

**adb tcpip 5555:** 设置设备监听 5555 端口上的 TCP/IP 的连接。

**adb connect xxx:**  通过目标设备的 IP 连接设备。

**adb disconnect ip:** 断开指定 IP 的设备 

### 基本操作

**adb devices:** 查找设备

**adb -s serialNum commond:** 指定设备执行命令

**adb install apk:** 安装 apk

**adb -s xxxx install apk:** 指定设备上安装 apk

**adb install -r apk:** 覆盖安装

**adb -d install apk:** 唯一 USB 连接设备安装 apk

**adb -e install apk:** 唯一模拟器设备安装 apk

**adb uninstall packageName:** 卸载 apk

**adb uninsatll -k packageName:** 卸载 apk,但是保留其配置和缓存文件

### 文件操作 

**adb push localFile remoteDictory:** 将本地的文件 push 远端指定的目录下

**adb pull file remoteDictory:** 从 remoteDictory 中复制指定的 file 到当前目录下

### adb 服务器

adb kill-server： 停止 adb 服务器

adb start-server：开启 adb 服务

### adb shell 

在目标设备启动远程 shell 


### adb shell am

使用 adb shell am 与应用交互。


|命令|含义|
--|--
start [option] <Intent>|启动 <Intent> 指定的 Activity
startservice [options] <INTENT>|	启动 <INTENT> 指定的 Service
broadcast [options] <INTENT>	|发送 <INTENT> 指定的广播
force-stop <packagename>|	停止 <packagename> 相关的进程

<Intent> 有关的选项
参数|含义
--|--
-a | <ACTION>	指定 action，比如 android.intent.action.VIEW
-c | <CATEGORY>	指定 category，比如 android.intent.category.APP_CONTACTS
-n | <COMPONENT>	指定完整 component 名，用于明确指定启动哪个 Activity，如 com.gy/.MainActivity

<Intent> 可以传参

```
adb shell start -n com.gy/.MainActivity //启动指定的 Activity

adb shell startService -n com.gy/.TestService //启动指定的 Service

adb shell broadcast  -a android.intent.action.BOOT_COMPLETED -n com.gy/.TestBroadcast //向指定的 BroadCast 发送广播

adb shell am force-stop com.gy // 关闭指定 app 的一切进程与服务
```

### adb shell pm 

### adb shell wm 
屏幕相关

```
adb shell wm size: 屏幕分辨率

adb shell wm size 480x1024: 屏幕分辨率修改为 480x1024

adb shell wm size reset: 恢复屏幕分辨率

adb shell wm density: 屏幕密度

adb shell wm density 160 : 屏幕密度设置为 160dpi

adb shell wm density reset: 恢复屏幕密度


```

### adb shell dumpsys 

查看运行状态