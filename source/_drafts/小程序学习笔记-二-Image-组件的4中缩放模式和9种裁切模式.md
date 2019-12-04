---
title: '小程序学习笔记(二): image 组件的4中缩放模式和9种裁切模式'
tags:
---


### 4 种缩放模式

通过 mode 指定缩放模式：

```
<image class="post-image" src="/images/post/post-4.jpg" mode="widthFix"/>
```

原图：

![](/source/images/2019_12_03_06.png)

 <!-- <img src="/source/images/2019_12_03_06.png" height="50%" width="50%"/> -->

* scaleToFill(默认)

![](/source/images/2019_12_03_07.png)，红色边框为边界，对比原图查看其缩放模式。

**不保持宽高比缩放图片**，使图片宽高完全拉伸至填满 iamge 组件。



* aspectFit
![](/source/images/2019_12_03_08.png)

**保持纵横比缩放图片**，使图片 **长边** 完全显示出来。

* aspectFill

![](/source/images/2019_12_03_09.png)


**保持纵横比缩放图片**，使图片 **短边** 完全显示出来。


* widthFix

![](/source/images/2019_12_03_10.png)

**宽度不变**，**高度自动变化**，**保持原图宽高比不变**。

此模式下将会改变 image 容器的高度。

### 9 种裁切模式

通过 mode 指定裁切模式：

```
<image class="post-image" src="/images/post/post-4.jpg" mode="top"/>
```

* top
  
![](/source/images/2019_12_03_11.png)

只保留图片的上部分。


其他不详述，大致一个意思。