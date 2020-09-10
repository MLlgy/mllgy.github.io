---
title: 记一次 Charles 抓包 Android 10.0 HTTPS 请求
date: 2020-07-30 13:48:15
tags: [Charles,HTTPS,Android]
---

Android 7.0 以后系统收紧了安全策略，其中一方面表现为：不再信任用户的 CA 证书，Android 官方文档作以下描述：

`默认情况下，面向 Android 7.0 的应用仅信任系统提供的证书，**且不再信任用户添加的证书颁发机构 (CA)**。如果面向 Android N 的应用希望信任用户添加的 CA，则应使用网络安全性配置以指定信任用户 CA 的方式。`

<!-- more -->

很明显按照文档，如果我们要信任 Charles 下发的 CA 证书，存在两个方案：

1. 按照文档所说，使用网络安全性配置以指定信任用户 CA。
2. 进行操作，使 Android 系统信任用户的 CA。

关于方案一自行查看官网文档 [网络安全性配置](https://developer.android.google.cn/training/articles/security-config),本文陈述如何将用户的 CA 证书变换为系统信任证书。

**前提：手机已 ROOT**

1. 如果之前信任过该证书，删掉；
2. `openssl x509 -inform PEM -subject_hash_old -in charles.crt | head -1`
   
   获得类似字符串：98f0087

3. 重命名 `mv charles.crt 98f0087.0`
4. 将 `98f0087.0` 移动到 `/system/etc/security/cacerts` 文件夹中；

至于最后一步，自己没有成功，原因是文件只读权限，即使使用 `mount -o remount -o rw /system` 无法更改其权限，自己进行的操作时将 `98f0087.0` push 手机端，通过 `Root Explore` 更改权限，并将该文件移至指定文件夹，至此就可以愉快的抓包 HTTPS 了。

