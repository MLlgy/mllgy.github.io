---
title: 减小 APK 体积
tags:
---






打包流程：

![](/source/images/2019_11_11_03.png)


从图上可知，引入的 Jar 包经过混淆，移除无用的代码，也会被打包到 dex 文件中。

### 包体积与应用性能


* 安装时间

文件拷贝、Library 解压、编译 ODEX、签名校验，特别对于 Android 5.0 和 6.0 系统来说（Android 7.0 之后有了混合编译），微信 13 个 Dex 光是编译 ODEX 的时间可能就要 5 分钟。

* 运行内存

Resource 资源、Library 以及 Dex 类加载这些都会占用不少的内存。

* ROM 空间

100MB 的安装包，启动解压之后很有可能就超过 200MB 了。对低端机用户来说，也会有很大的压力。如果闪存空间不足，非常容易出现写入放大的情况。


### Dex 分包导致包体积增大

使用 AS 查看 APK 的 dex 文件时，可以看到如下所示的信息：

![](/source/images/2019_11_21_02.png)


“define classes and methods”是指真正在这个 Dex 中定义的类以及它们的方法。而“reference methods”指的是 define methods 以及 define methods 引用到的方法。

举一个浅显的例子：如果将 Class A 与 Class B 分别编译到不同的 Dex 中，由于 method a 调用了 method b，所以 classes2.dex 文件中也需要加上 method b 的id，如下图所示：

![](/source/images/2019_11_21_03.png)

如图所示，由于跨 Dex 调用的存在，导致有些 Dex 文件中存在一些冗余信息，比如图中 classes2.dex  中的 method id。此现象的存在对 Dex 的大小的影响：

* method id 爆表

我们都知道每个 Dex 的 method id 需要小于 65536，**因为 method id 的大量冗余导致每个 Dex 真正可以放的 Class 变少**，这是造成最终编译的 Dex 数量增多。
 
* 冗余信息

因为我们 **需要记录跨 Dex 调用的方法的详细信息**，所以在 classes2.dex 我们还需要记录 Class B 以及 method b 的定义，造成 string_ids、type_ids、proto_ids 这几部分信息的冗余。


很明显提升 Dex 中 define methods 中的比重，会减少 dex 的体积，那么 APK 的体积也会减少。那么如何提高 Dex 中 define methods 中的比重呢？**将有调用关系的类和方法分配到同一个 Dex，即减少跨 Dex 调用的情况**。但是针对此的优化只能达到局部最优解，因为需要在编译速度和效果之间找一个平衡点。

### 压缩代码


**1. 使用 minifyEnabled 属性**

开启 minifyEnabled true 属性，会进行无用代码的裁切。

**2. 进行混淆**

除了启用 minifyEnabled 属性，还可以在 `proguard-rules.pro` 文件中配置混淆规则，把相关类和方法混淆成短路径，比如混淆方法：

```
void show()  -> void a()
```

将类和方法混淆成短路径，那么构建的 Dex 文件的大小也会减小。


### 移除无用的资源

Android 官方早就为我们考虑好了，以下为关于无用资源优化方案的演进：

**第一阶段: Lint**

Lint 是一款静态代码扫描工具，其中有一项关于 Unused Resources 的扫描，然后直接选择 `Remove All Unused Resources`，这样就可以直接删除所有无用的资源。

但是 Lint 作为第一阶段的方案，在扫描过程中存在一些缺点：

由于 Lint 是静态代码扫描工具，其最大的问题是没有考虑到 ProGuard 的代码裁剪，在 ProGuard 过程中会 shrink 掉大量的无用代码，但是 Lint 工具并不会检查出这些无用代码引用的无用资源。

当然 Lint 还有很多其他的特色功能：检查布局嵌套过深、存在的内存泄漏隐患等。

**第二阶段：shrinkResources**

基于 Lint 的缺点，在第二阶段 Android 增加了 shrinkResources 的资源压缩功能，但是它需要和 ProGuard 的 minifyEnabled 功能配合使用，启用 minifyEnabled 功能会将部分无用的代码移除，而这些代码引用的资源也会被标记为无用资源，开启 shrinkResources 通过资源压缩功能将它们移除。

要使用 shrinkResources，您必须启用代码压缩功能。在编译过程中，首先，ProGuard 会移除未使用的代码，但会保留未使用的资源。然后，Gradle 会移除未使用的资源

```

android {
    ...
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
        }
    }
}
```
但是目前的 shrinkResources 在实现上还是存在以下缺陷：

1. 没有处理 resources.arsc 文件，导致大量无用的 String、Id、Attr、Dimen、Color 等资源并没有被删除。

![](/source/images/2019_11_07_01.png)

如上图，以 umeng 开头的 color 资源其实在应用中并没有被使用，但是还是存在于 resources.arsc。

2. 没有真正的删除资源文件。

对于 Drawable 、Layout 这些无用资源，shrinkResources 也没有把它们真正删掉，而仅仅替换为一个空文件，为什么不删除呢？由于 没有处理 resources.arsc 文件，所以 resources.arsc 中还有这些文件的路径。

![](/source/images/2019_11_07_02.png)

如上图，标记的文件在 Apk 中被替换为 空文件。


同时在生成的 resources.txt 文件中也可以看到该文件被替换为空文件的的日志：
![](/source/images/2019_11_07_03.png)

同时也存在对无用的资源文件不做任何处理的情况，如下图所示：

![](/source/images/2019_11_07_04.png)

可以看到 activity_edit_address.xml 在文件中没有被引用，该文件在 Apk 构建过程中也没有被替换为空文件，到 resources.txt 文件中去查看该文件的信息：

![](/source/images/2019_11_07_05.png)

那么为什么会出现这种情况呢，具体可以看 [ResourceUsageAnalyzer.java](https://android.googlesource.com/platform/tools/base/+/studio-master-dev/build-system/gradle-core/src/main/java/com/android/build/gradle/tasks/ResourceUsageAnalyzer.java) 中的相关代码：

```
if (mFoundWebContent) {
                Resource resource = mModel.getResourceFromFilePath(string);
                if (resource != null) {
                    ResourceUsageModel.markReachable(resource);
                    continue;
                } else {
                    int start = 0;
                    int slash = string.lastIndexOf('/');
                    if (slash != -1) {
                        start = slash + 1;
                    }
                    int dot = string.indexOf('.', start);
                    String name = string.substring(start, dot != -1 ? dot : string.length());
                    if (names.contains(name)) {
                        for (Map<String, Resource> map : mModel.getResourceMaps()) {
                            resource = map.get(name);
                            if (mDebug && resource != null) {
                                mDebugPrinter.println("Marking " + resource + " used because it "
                                        + "matches string pool constant " + string);
                            }
                            ResourceUsageModel.markReachable(resource);
                            // 加入白名单
                            mModel.addResourceToWhitelist(resource);
                        }
                    }
                }
            }

```

大致意思资源文件名的前缀包含在 string pool 中，那么该文件会被包含在白名单中，在资源压缩、移除过程中，不会被置为空文件或者删除，在该项目中以 umeng 开头的 Layout 文件也不会被置空或删除，但是 string pool 是如何初始化的，自己没有进一步阅读源码，还不清楚，有空仔细阅读。

针对此种情况，可以先使用 Lint 工具对项目进行 Unused Resources 的扫描，并且移除。

**第三阶段：realShrinkResources**

目前 Android 还没有提供这套方案的实现，故不详述。


----

[压缩、混淆和优化您的应用](https://developer.android.google.cn/studio/build/shrink-code.html#top_of_page)，在 outputs/mapming 文件夹下会生成以下文件：

* dump.txt:描述apk文件中所有类文件间的内部结构.
* mapping.txt:列出了混淆过的类、方法和字段名称与原始名称的映射关系，能够通过该文件进行解码混淆过的堆栈轨迹，比如上传到 友盟统计平台，显示 bug 的堆栈轨迹。
* seeds.txt:项目的保留规则确定的入口点的报告(列出未进行混淆的类和成员)。
* usage.txt:移除的代码的报告。
* resources.txt：此文件包含一些详细信息，如哪些资源引用了其他资源以及使用或移除了哪些资源。


----

[缩减应用大小](https://developer.android.google.cn/topic/performance/reduce-apk-size)

[压缩、混淆和优化您的应用](https://developer.android.google.cn/studio/build/shrink-code.html#top_of_page):


[使用 APK 分析器分析您的编译版本](https://developer.android.google.cn/studio/build/apk-analyzer?hl=zh_cn)


[压缩您的应用](https://developer.android.google.cn/studio/build/shrink-code?hl=zh_cn)








所以尽管我们的应用中仍旧存在大量的无用资源，但是系统目前的做法并没有真正减少文件数量。这样 resources.arsc、签名信息以及 ZIP 文件信息这几个“大头”依然没有任何改善。



> resources.arsc、R.java 文件的资源 ID 是连续的。





![](/source/images/2019_11_06_01.png)


此处的 APK 为 build 生产的包


### Lint 

**Unuse resource**

![](/source/images/2019_11_06_02.png)



### 移除未使用的语言以及 so 库

如果您希望只保留应用正式支持的语言，可以使用 `resConfig` 属性指定这些语言。系统会移除未指定语言的所有资源。

```
android {
    defaultConfig {
        resConfigs "en", "zh"
        // 只保留 armeabi 下的 so 库
        ndk{
            abiFilters 'armeabi'
        }
    }
}
```
关于 Android CPU 架构的基础知识可以查看 [Android CPU 架构](https://leegyplus.github.io/2019/11/12/Android-CPU-%E6%9E%B6%E6%9E%84/)，正如该文所说， armeabi 指令集能够兼容其他大部分 CPU 架构，但是也是有一定代价的，比如文中所说的会生成相应的模拟层，相应的会对性能产生一定的影响。

有些大佬是强烈反对仅仅为了减少 APK 体积而仅仅保留 armeabi 的 so 文件，比如刘皇叔在 [关于Android的.so文件你所需要知道的](https://www.jianshu.com/p/cb05698a1968) 中的观点，但是大部份针对 APK 体积的优化还是都会采取此措施，然后针对性测试，毕竟其优化效果是十分显著的。如果引入的库(手动添加的 so 库 + 三方依赖中的 so 库)较多的话，基本上可以得到 M 级别的优化空间。


项目中 so 库的来源：

* 在项目中手动添加的 so 库
* 三方依赖库中依赖的 so 库

在上面的 apk 打包流程图中也可以看到，此图真实细致入微。 


----


## 1、代码层面

### ProGuard

是否存在过度keep 的现象。



----

blogs

https://juejin.im/post/5d4407baf265da03f04caf59

[官方文档：压缩、混淆和优化您的应用](https://developer.android.google.cn/studio/build/shrink-code.html)

[缩减应用大小](https://developer.android.google.cn/topic/performance/reduce-apk-size)



[Android 中资源文件概览](https://developer.android.google.cn/guide/topics/resources/providing-resources.html#AlternativeResources)



[Android APP终极瘦身指南](http://jayfeng.com/2016/03/01/Android-APP%E7%BB%88%E6%9E%81%E7%98%A6%E8%BA%AB%E6%8C%87%E5%8D%97/) -- 指明了多个条例






1. 移除无用的资源
2. 对图片的处理：.9 图、webp、一套图、压缩(也许这是效果最明显的)




----


根据解压后的 Apk 的各个文件，可以看到占用空间最大的是代码和资源，那么减小 Apk 体积需要着重从这两方面出发。

**代码部分：**

    冗余代码、无用功能、代码混淆、方法数缩减

**资源部分：**

    冗余资源、资源混淆、图片处理(压缩、图片转换、点9图化等)

**对整个安装包做 7zip 极限压缩**

---
原始状态：无混淆

|状态|包大小|lib|res|dex|classes.dex|classes2.dex|assets|resources.arsc|META_INFO|publicxx.gz|AndroidManifest.xml|总结
|--|--|--|--|--|--|--|--|--|--|--|--|--|
原始状态|19.9M|7.1M|5.8M|4.6M|3.5M|1.1M|1.3M|766.5K|108.7K|33.2K|5.9K|
|使用 Lint 移除无用资源|19.1M(-0.8M)|7.1M|5.1M(-0.7M)|4.6M|3.5M|1.1M|1.3M|667.9K(-98.6K)|94.6K|33.2K|5.9K|减少资源文件，res 中的文件减少，同时 resources.arsc 中的字节码也会减少
|ProGuard 开启代码混淆及裁剪|18.4M(-0.7M)|7.1M|5.1M|3.91M(-0.69M)|3.6M|327.3K|1.3M|667.9K|94.7K|33.2K|5.9K|文件路径及方法名变短，那么 .dex 文件大小减小
| ProGuard 开启资源混淆进行压缩|18.3M(-0.1M)|同上|同上|同上|同上|同上|同上|同上|94.2K|同上|同上|


以上为通过 Lint 移除无用资源、ProGuard 代码混淆以及资源移除的 Apk 的变化，从以上图表可以看出对资源文件的处理可以大幅度的减少 Apk 的大小，那么还有什么方案可以对资源文件进一步处理？


* 图片文件格式由 PNG 到 JPG 的转换

由于 PNG 为无损格式， JPG 为有损格式，JPG 在处理时会根据压缩率的不同会去掉部分识别差别较小的中间颜色，而 PNG 会严格保留所有的色彩，所以在尺寸大、色彩多的时候 PNG 的体积会明显大于 JPG。可以在小尺寸、色彩少，或者有 alpha 透明度通道的时候使用 PNG，在大尺寸、渐变色多的时候用 JPG，不过这一点需要跟设计师协商后决定。但是在可以使用 JPG 格式的图片情况下，最好不要从 PNG 转到 JPG，最好让设计师给出相应的图片，效果会更好一些。

* 图片压缩

对于 APK 来说，大于 5K 的图片就算是大的图片了，所以 95% 的图片应该小于 5K，在 APK 打包过程中，aapt 工具会采用 [crunch](https://github.com/Unity-Technologies/crunch/tree/unity) 对图片做预处理，但是其压缩率并不是最好的，采用 [pnggauntlet](https://pnggauntlet.com/#more) 对非 .9 图片进行压缩，可以得到更好的压缩率，而使用该工具对 .9 图进行压缩的话会有黑边，视觉效果不好。

下面分别采用两种方式对同一个图片进行压缩：

* crunch

相关的 aapt 描述如下：
> aapt c[runch] [-v] -S resource-sources ... -C output-folder ...

Do PNG preprocessing on one or several resource folders and store the results in the output folder.(对目标文件夹中的 PNG 图片进行预处理，并将处理结果存放在相应目录中。)


使用 aapt 命令，采用 crunch 的图片进行预处理：

> aapt c -S crunch -C crunch_1

其中 crunch 为原始图片所在的目录，crunch_1 为处理后输出的文件目录。

以下为处理结果：

![](/source/images/2019_11_11_01.png)

* pnggauntlet 

使用 pnggauntlet 对图片进行压缩需要下载相应的软件，不过注意的是该工具会直接在原图片的基础上做压缩，所以应该做好图片的备份，防止意外情况。

![](/source/images/2019_11_11_02.png)




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

AndResGuard 的两个功能：资源混淆、资源的极限压缩。

[安装包立减1M--微信Android资源混淆打包工具](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=208135658&idx=1&sn=ac9bd6b4927e9e82f9fa14e396183a8f#rd)

[如何使用资源混淆工具](https://github.com/shwenzhang/AndResGuard/blob/master/doc/how_to_work.zh-cn.md)

[Android Apk 7z压缩](http://blog.rcant.com/2017/03/19/others/android-andresguard/)



* Dex 压缩

Redex

* Library 压缩


|状态|包大小|lib|res|dex|classes.dex|classes2.dex|assets|resources.arsc|META_INFO|publicxx.gz|AndroidManifest.xml|总结
|--|--|--|--|--|--|--|--|--|--|--|--|--|
| ProGuard 开启资源混淆进行压缩|18.3M(-0.1M)|7.1M|5.1M|3.91M(-0.69M)|3.6M|327.3K|1.3M|667.9K|94.2K|33.2K|5.9K|
|取消默认的 crunch 预处理过程|20.1M|7.1M|6.8M|3.9M|3.6M|327.3K|1.3M|667.9K|94.2K|33.2K|5.9K|由于取消 crunch 预处理，所以 APK 包增大
| 使用 pnggaunlet 工具对图片资源进行处理|15.4M|7.1MB|2.1M|3.9M|3.6M|327.3K|1.3M|667.9K|94.2K|33.2K|5.9K|减小了图片的大小，res 文件夹大小当然会减小
开启 crunch 预处理，apk 大小几乎无变化|15.4M|7.1MB|2.1M|3.9M|3.6M|327.3K|1.3M|667.9K|94.2K|33.2K|5.9K|由于使用 pnggaunlet 获得更大的压缩率，此时开启 crunch 几乎无压缩效果
|仅保留 armeabi 中的 so 库|10.4M|2.2MB|2.1M|3.9M|3.6M|327.3K|1.3M|667.9K|93.6K|33.2K|5.9K|该优化措施后，进行针对性测试，在自己的测试机上 App 正常运行
使用 AndResGuard 进行资源混淆和资源的极限压缩||||||||||||在经过之前的步骤后，图片结果压缩操作，此操作对资源的极限压缩空间不大， APK 大小几乎不变(大约级别为百K 级别)，在对原始APK 进行此操作，APK 包体积大约减小 1 M|

* 图片 .9 化

点9图是 Android 平台开发的一种特殊的图片格式，扩展名为 .9.png。点9图相当于把一张 png 图片分成了 9 部分，

|1|2|3|
--|--|--
4|5|6|
7|8|9

四个角的位置(1、3、7、8)是不做拉伸的，两条水平部分(2、8)只做水平拉伸，两条垂直部分(4、6)只做垂直拉伸，所以不会出现四条边被拉粗的情况，而中间部分(5)用黑线指定区域进行拉伸。

* 删除项目中不会再用到的图片

比如某个布局的 visiable 为 GONE状态，并且项目中没有显示该布局的需求，那么就可以就该布局删除，并且将只在该布局引用的图片资源删除，还有其他类似的情况。



----





----


[Shrinking Your Android App Size](https://devblogs.microsoft.com/xamarin/shrinking-android-app-size/)


[Shrinking Your App with R8](https://v.youku.com/v_show/id_XNDQxMzE4MTgyOA==.html?spm=a2hzp.8253869.0.0&utm_source=androidweekly.io&utm_medium=website)


[APK瘦身记，如何实现高达53%的压缩效果](https://www.cnblogs.com/alisecurity/p/5341218.html)


[Android拆分与加载Dex的多种方案对比](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207151651&idx=1&sn=9eab282711f4eb2b4daf2fbae5a5ca9a&3rd=MzA3MDU4NTYzMw==&scene=6#rd)

[抖音包大小优化：资源优化](https://www.infoq.cn/article/ITVEfvGD5Uv6r07NHssx)