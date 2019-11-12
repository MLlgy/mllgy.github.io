---
title: FileProvider 相关
tags:
---




一个应用程序可以向另外一个应用程序提供一个或多个文件，例如：获得图库中的照片进行编辑、读取 Apk 进行并且安装等。但是以上功能的实现在 Android 7.0 时进行了变更，具体可以查看 [谷歌官方说明:在应用间共享文件](https://developer.android.google.cn/about/versions/nougat/android-7.0-changes#sharing-files)，在 Android 7.0 以后禁止在应用中公开使用 file:// URI，否则会出现 FileUriExposedException 异常。Android7.0 后，在应用间共享文件要使用 content:// URI,并授予 URI 临时权限





https://www.cnblogs.com/newjeremy/p/7294519.html


https://blog.csdn.net/github_2011/article/details/78589514

https://www.jianshu.com/p/121bbb07cb07


[Android 7.0 行为变更 通过FileProvider在应用间共享文件吧](https://blog.csdn.net/lmj623565791/article/details/72859156)

