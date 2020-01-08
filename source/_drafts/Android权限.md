---
title: Android权限
tags:
---

清单文件的 

normal permission：系统自动授权。
dangerous permission：需要用户显示授权。


授权的时机：

6.0 及以上：>=23

运行时提示授权


5.0 及以下：<= 22

安装时提示，如果拒绝，在系统终止安装 app

应用升级，如果包含新的权限，则在更新前会提示对新的权限进行授权。


### 可选硬件功能的权限


比如蓝牙、相机。

但是并不是所有的设备都拥有这些硬件功能，如果应用需要相机权限，则在清单文件中，对相机权限的声明应该使用 <uses-feature> 标签：


```
<uses-feature android:name="android.hardware.camera" android:required="false" />
```

但是如果在应用中不提供 <uses-feature>，但是 Goole Play 检测应用程序请求相应的授权时，那么此时就和声明该标签一样，会对没有该特性的设备进行过滤。


### 查看 app 的权限

adb shell pm list permission -s