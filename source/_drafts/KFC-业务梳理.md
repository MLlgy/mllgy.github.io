---
title: KFC 业务梳理
tags:
---


### 1. Android 中打开微信小程序

添加依赖：

```
 implementation 'com.tencent.mm.opensdk:wechat-sdk-android-without-mta:+'
```

WideShareService#openWechatMini

回调类：

WXEntryActivity
WXPayEntryActivity

[官方文档](https://developers.weixin.qq.com/doc/oplatform/Mobile_App/Launching_a_Mini_Program/Android_Development_example.html)