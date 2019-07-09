---
title: LeakCanary1.x 是如何工作的
tags:
---


### 0x0001 
监听对象

Activity、Activity.view Fragement Fragment.view

refWatch.add(object)


#### 0x0002
步骤 ：
安装
监听
分析
显示

### 0x0003
安装








> 科普小知识
hprof 文件：
> A heap dump is a snapshot of all the objects that are in memory in the JVM at a certain moment. They are very useful to troubleshoot memory-leak problems and optimize memory usage in Java applications. **Heap dumps are usually stored in binary format hprof files.** 



----

[WeakRefenerce 的理解和使用](https://blog.csdn.net/zmx729618/article/details/54093532)

https://mp.weixin.qq.com/s/uwDk5D986OdMzKgtjpHdHg

https://mp.weixin.qq.com/s/idjFaJsLpVLw52RSYHA_Vg




[LeakCanary 1.x 源码原理](https://mp.weixin.qq.com/s/idjFaJsLpVLw52RSYHA_Vg)