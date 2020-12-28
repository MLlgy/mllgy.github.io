---
title: Android 多媒体库相关问题
tags:
---


### Android 默认行为

Android 默认会将每个多媒体的文件保存在一个数据库中，用户的一些行为会触发系统扫描文件系统的多媒体文件变化情况并同步到多媒体数据库中。

如果不希望文件夹中的多媒体文件保存到多媒体数据库中，在该文件夹下新建 .nomedia 文件，那么系统在更新媒体数据库时会忽略这个文件夹。


---

**知识链接：**

[Android通过.nomedia文件禁止多媒体库扫描指定文件夹下的多媒体文件](https://zhuanlan.zhihu.com/p/20851896)

[Android扫描多媒体文件剖析](https://droidyue.com/blog/2014/07/12/scan-media-files-in-android-chinese-edition/)