---
title: LeakCanary1.x 是如何工作的
tags:
---


### 0x0001 
监听对象

Activity、Activity.view Fragement Fragment.view

refWatch.watch(object)(可以自定义添加监听对象吗)


添加检测对象的逻辑：

以 Activity 为例，
首先在 Activity 被销毁后(执行 destory())，此时 LeakCanary 会此时会触发 LeakCanary 的检测引用可达性，如果一个引用为不可达状态




#### 0x0002
步骤 ：
安装
监听
分析
显示

### 0x0003
安装









----

[WeakRefenerce 的理解和使用](https://blog.csdn.net/zmx729618/article/details/54093532)

https://mp.weixin.qq.com/s/uwDk5D986OdMzKgtjpHdHg

https://mp.weixin.qq.com/s/idjFaJsLpVLw52RSYHA_Vg




[LeakCanary 1.x 源码原理](https://mp.weixin.qq.com/s/idjFaJsLpVLw52RSYHA_Vg)


[LeakCanary实战](https://www.sunmoonblog.com/2018/02/02/leakcanary-application/)