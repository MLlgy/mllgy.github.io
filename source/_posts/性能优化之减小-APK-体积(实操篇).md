---
title: 性能优化之包体积优化(实操篇)
date: 2019-11-06 15:42:58
tags: [Android 性能优化]
---


根据解压后的 Apk 的各个文件，可以看到占用空间最大的是代码和资源，那么减小 Apk 体积需要着重从这两方面出发。

**代码部分：**

    冗余代码、无用功能、代码混淆、方法数缩减

**资源部分：**

    冗余资源、资源混淆、图片处理(压缩、图片转换、点9图化等)
<!-- more -->

**对整个安装包做 7zip 极限压缩**

    针对 7zip 的操作可以参见微信方案：[AndResGuard](https://github.com/shwenzhang/AndResGuard)


## 0x0000 代码层面

### ProGuard

是否存在过度keep 的现象。

以下表格为包体积的改变：

|状态|包大小|lib|res|dex|classes.dex|classes2.dex|assets|resources.arsc|META_INFO|publicxx.gz|AndroidManifest.xml|总结
|--|--|--|--|--|--|--|--|--|--|--|--|--|
原始状态(无混淆)|19.9M|7.1M|5.8M|4.6M|3.5M|1.1M|1.3M|766.5K|108.7K|33.2K|5.9K|
|使用 Lint 移除无用资源|19.1M(-0.8M)|7.1M|5.1M(-0.7M)|4.6M|3.5M|1.1M|1.3M|667.9K(-98.6K)|94.6K|33.2K|5.9K|减少资源文件，res 中的文件减少，同时 resources.arsc 中的字节码也会减少
|ProGuard 开启代码混淆及裁剪|18.4M(-0.7M)|7.1M|5.1M|3.91M(-0.69M)|3.6M|327.3K|1.3M|667.9K|94.7K|33.2K|5.9K|文件路径及方法名变短，那么 .dex 文件大小减小
| ProGuard 开启资源混淆进行压缩|18.3M(-0.1M)|同上|同上|同上|同上|同上|同上|同上|94.2K|同上|同上|


以上为通过 Lint 移除无用资源、ProGuard 代码混淆以及资源移除的 Apk 的变化，从以上图表可以看出对资源文件的处理可以大幅度的减少 Apk 的大小，那么还有什么方案可以对资源文件进一步处理？

## 0x0001 代码层面

### 图片文件格式由 PNG 到 JPG 的转换

由于 PNG 为无损格式， JPG 为有损格式，JPG 在处理时会根据压缩率的不同会去掉部分识别差别较小的中间颜色，而 PNG 会严格保留所有的色彩，所以在尺寸大、色彩多的时候 PNG 的体积会明显大于 JPG。可以在小尺寸、色彩少，或者有 alpha 透明度通道的时候使用 PNG，在大尺寸、渐变色多的时候用 JPG，不过这一点需要跟设计师协商后决定。但是在可以使用 JPG 格式的图片情况下，最好不要从 PNG 转到 JPG，最好让设计师给出相应的图片，效果会更好一些。

### 图片压缩

对于 APK 来说，大于 5K 的图片就算是大的图片了，所以 95% 的图片应该小于 5K，在 APK 打包过程中，aapt 工具会采用 [crunch](https://github.com/Unity-Technologies/crunch/tree/unity) 对图片做预处理，但是其压缩率并不是最好的，采用 [pnggauntlet](https://pnggauntlet.com/#more) 对非 .9 图片进行压缩，可以得到更好的压缩率，而使用该工具对 .9 图进行压缩的话会有黑边，视觉效果不好。

下面分别采用两种方式对同一个图片进行压缩：

* crunch
  
使用命令对资源文件进行压缩： `aapt c[runch] [-v] -S resource-sources ... -C output-folder ...`

相关的 aapt 描述如下：


    Do PNG preprocessing on one or several resource folders and store the results in the output folder.(对目标文件夹中的 PNG 图片进行预处理，并将处理结果存放在相应目录中。)


使用 aapt 命令，采用 crunch 的图片进行预处理：

> aapt c -S crunch -C crunch_1

其中 crunch 为原始图片所在的目录，crunch_1 为处理后输出的文件目录。

以下为处理结果：

![](性能优化之减小-APK-体积/2019_11_11_01.png)

* pnggauntlet 

使用 pnggauntlet 对图片进行压缩需要 **下载相应的软件**，不过注意的是该工具会直接在原图片的基础上做压缩，所以应该做好图片的备份，防止意外情况。

![](性能优化之减小-APK-体积/2019_11_11_02.png)


以下针对 crunch、pnggauntlet 的压缩率进行测试，首先以上文中处理后的 APk 为基础进行对照。

1. 由于在 APK 中的会默认使用 crunch 进行预处理，那么首先我们关闭该预处理项，观察 APK 会增大多少。

关闭 crunch 预处理过程
```
buildTypes {
    release {
        ...
        crunchPngs false
    }
}
```


2. 使用 pnggaunlet 工具对图片资源进行压缩。默认情况下，该工具开启的为质量压缩(100%)，也可以开启 pnggaunlet 的有损压缩，设置相应的压缩比，这样可以获得更大的压缩率，但是可能会对图片造成影响。
   
项目中居然存在由几百 K 的图片，多次压缩后不能继续优化。针对此时的情景，应该同 UI 设计师协商，综合多方面考虑能否给出更小的图片。

同时，还存在其他的优秀的图片压缩工具，比如 [Tinypng](https://tinypng.com/) 等，可以根据需求进行相应的选择。


* AndResGuard 资源的资源混淆和极限压缩

AndResGuard 的两个功能：资源混淆、资源的极限压缩，具体可以参考链接：[安装包立减1M--微信Android资源混淆打包工具](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=208135658&idx=1&sn=ac9bd6b4927e9e82f9fa14e396183a8f#rd)、[如何使用资源混淆工具](https://github.com/shwenzhang/AndResGuard/blob/master/doc/how_to_work.zh-cn.md)、[Android Apk 7z压缩](http://blog.rcant.com/2017/03/19/others/android-andresguard/)



### Dex 压缩

Redex

### Library 压缩


**使用不同方法减少包体积以及具体效果**

|状态|包大小|lib|res|dex|classes.dex|classes2.dex|assets|resources.arsc|META_INFO|publicxx.gz|AndroidManifest.xml|总结
|--|--|--|--|--|--|--|--|--|--|--|--|--|
| ProGuard 开启资源混淆进行压缩|18.3M(-0.1M)|7.1M|5.1M|3.91M(-0.69M)|3.6M|327.3K|1.3M|667.9K|94.2K|33.2K|5.9K|
|取消默认的 crunch 预处理过程|20.1M|7.1M|6.8M|3.9M|3.6M|327.3K|1.3M|667.9K|94.2K|33.2K|5.9K|由于取消 crunch 预处理，所以 APK 包增大
| 使用 pnggaunlet 工具对图片资源进行处理|15.4M|7.1MB|2.1M|3.9M|3.6M|327.3K|1.3M|667.9K|94.2K|33.2K|5.9K|减小了图片的大小，res 文件夹大小当然会减小
开启 crunch 预处理，apk 大小几乎无变化|15.4M|7.1MB|2.1M|3.9M|3.6M|327.3K|1.3M|667.9K|94.2K|33.2K|5.9K|由于使用 pnggaunlet 获得更大的压缩率，此时开启 crunch 几乎无压缩效果
|仅保留 armeabi 中的 so 库|10.4M|2.2MB|2.1M|3.9M|3.6M|327.3K|1.3M|667.9K|93.6K|33.2K|5.9K|该优化措施后，进行针对性测试，在自己的测试机上 App 正常运行
使用 AndResGuard 进行资源混淆和资源的极限压缩||||||||||||在经过之前的步骤后，图片结果压缩操作，此操作对资源的极限压缩空间不大， APK 大小几乎不变(大约级别为百K 级别)，在对原始APK 进行此操作，APK 包体积大约减小 1 M|

### 图片 .9 化

点9图是 Android 平台开发的一种特殊的图片格式，扩展名为 .9.png。点9图相当于把一张 png 图片分成了 9 部分，

|1|2|3|
--|--|--
4|5|6|
7|8|9

四个角的位置(1、3、7、8)是不做拉伸的，两条水平部分(2、8)只做水平拉伸，两条垂直部分(4、6)只做垂直拉伸，所以不会出现四条边被拉粗的情况，而中间部分(5)用黑线指定区域进行拉伸。

### 删除项目中不会再用到的图片

比如某个布局的 visiable 为 GONE状态，并且项目中没有显示该布局的需求，那么就可以就该布局删除，并且将只在该布局引用的图片资源删除，还有其他类似的情况。

----

**知识链接**

[Shrinking Your Android App Size](https://devblogs.microsoft.com/xamarin/shrinking-android-app-size/)

[Shrinking Your App with R8](https://v.youku.com/v_show/id_XNDQxMzE4MTgyOA==.html?spm=a2hzp.8253869.0.0&utm_source=androidweekly.io&utm_medium=website)

[APK瘦身记，如何实现高达53%的压缩效果](https://www.cnblogs.com/alisecurity/p/5341218.html)

[Android拆分与加载Dex的多种方案对比](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207151651&idx=1&sn=9eab282711f4eb2b4daf2fbae5a5ca9a&3rd=MzA3MDU4NTYzMw==&scene=6#rd)

[抖音包大小优化：资源优化](https://www.infoq.cn/article/ITVEfvGD5Uv6r07NHssx)

[Android 可能你想要的APK瘦身笔记](https://juejin.im/post/5d4407baf265da03f04caf59)

[官方文档：压缩、混淆和优化您的应用](https://developer.android.google.cn/studio/build/shrink-code.html)

[缩减应用大小](https://developer.android.google.cn/topic/performance/reduce-apk-size)

[Android 中资源文件概览](https://developer.android.google.cn/guide/topics/resources/providing-resources.html#AlternativeResources)

[Android APP终极瘦身指南](http://jayfeng.com/2016/03/01/Android-APP%E7%BB%88%E6%9E%81%E7%98%A6%E8%BA%AB%E6%8C%87%E5%8D%97/) -- 指明了多个条例
