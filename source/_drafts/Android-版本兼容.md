---
title: Android 版本兼容
tags:
---


### 6.0(API 23)


* 运行时权限适配

权限组问题：在 6.0 ~ 8.0，同一权限组中，只要有一个被同意，其他的就会被同意。而在 8.0 之后，此行为已被纠正。系统只会授予应用明确请求的权限。然而一旦用户为应用授予某个权限，则所有后续对该权限组中权限的请求都将被自动批准，但是若没有请求相应的权限而进行操作的话就会出现应用 crash 的情况。

* 低电耗模式和应用待机模式

* 取消支持 Apache HTTP 客户端

 Android 2.3（API 级别 9）或更高版本为目标平台，请改用 HttpURLConnection，此 API 效率更高，因为它可以通过透明压缩和响应缓存减少网络使用，并可最大限度降低耗电量。要继续使用 Apache HTTP API，您必须先在 build.gradle 文件中声明以下编译时依赖项：
```
android {
    useLibrary 'org.apache.http.legacy'
}
```
* 更换使用通知的方式

此版本移除了 Notification.setLatestEventInfo() 方法。请改用 Notification.Builder 类来构建通知。要重复更新通知，请重复使用 Notification.Builder 实例。调用 build() 方法可获取更新后的 Notification 实例。

[Android 6.0 变更](https://developer.android.google.cn/about/versions/marshmallow/android-6.0-changes)

### 7.0(API 24、Nougat)
1. 针对电池和内存

    * 低电耗模式(非静止状态下也会进入低电耗)
    * Project Svelte：后台优化(Android 7.0 移除了三项隐式广播,以帮助优化内存使用和电量消耗)


2. 权限更改

    系统权限更改,为了提高私有文件的安全性，面向 Android 7.0 或更高版本的应用私有目录被限制访问　(0700).分享私有文件内容的推荐方法是使用 FileProvider

3. 在应用间共享文件

使用 FileProvider


### Android 8.0 (Oreo、API26)



1. 针对所有 API 级别的应用


后台执行限制、Android 后台位置限制

权限
在 Android 8.0 之前，如果应用在运行时请求权限并且被授予该权限，系统会错误地将属于同一权限组并且在清单中注册的其他权限也一起授予应用。

对于针对 Android 8.0 的应用，此行为已被纠正。系统只会授予应用明确请求的权限。然而，一旦用户为应用授予某个权限，则所有后续对该权限组中权限的请求都将被自动批准。


2. 通知

[Android通知栏微技巧，8.0系统中通知栏的适配](https://blog.csdn.net/guolin_blog/article/details/79854070)

3. 应用图标

[Android应用图标微技巧，8.0系统中应用图标的适配](https://blog.csdn.net/guolin_blog/article/details/79417483)


### Android 9.0 

[Android 9.0系统新特性，对刘海屏设备进行适配](https://blog.csdn.net/guolin_blog/article/details/103112795)

---

[Android版本适配(基于 6.0 ~ 9.0)](https://www.jianshu.com/p/46699e98dc59)

[Android 版本适配问题](https://www.jianshu.com/p/74b43dd828b0)

[Android targetSdkVersion 版本适配指北（持续更新）](https://johnnyshieh.me/posts/android-target-sdk-version/)

[Android 8.0 适配](https://www.jianshu.com/p/d9f5b0801c6b)