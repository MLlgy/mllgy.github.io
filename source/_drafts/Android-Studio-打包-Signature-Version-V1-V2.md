---
title: Android Studio 打包 Signature Version V1/ V2
tags:
---


![项目截图](https://upload-images.jianshu.io/upload_images/1969719-dd4578f0e7280a5d.png?imageMogr2/auto-orient/strip|imageView2/2/w/581/format/webp)

在上面的截图中，我们可以看到 Signature Version 有两个选择
* V1 (jar Signature)
* V2 (Full APK Signature)
  
### 0x0001  两者的区别

* 解释一
  
    V1: Jar Signature 来自 JDK，可对签名后的文件，作适当修改，并重新压缩

    V2: Android 7.0 (Nougat) 引入的一项新的签名方案，不能对签名后的 APK作任何修改，包括重新解压。因为它是针对字节进行的签名，所以任何改动都会影响最终结果。
* 解释二


    Android 7.0中引入了APK Signature Scheme v2，v1 是jar Signature来自JDK

    V1：应该是通过ZIP条目进行验证，这样APK 签署后可进行许多修改 - 可以移动甚至重新压缩文件。

    V2：验证压缩文件的所有字节，而不是单个 ZIP 条目，因此，在签名后无法再更改(包括 zipalign)。正因如此，现在在编译过程中，我们将压缩、调整和签署合并成一步完成。好处显而易见，更安全而且新的签名可缩短在设备上进行验证的时间（不需要费时地解压缩然后验证），从而加快应用安装速度。


### 0x0002 如何使用

* 方式一：在打包时选择指定签名方式，如文首图片：

  * 只勾选v1签名所有机型都能用，但是在7.0及以上不会使用更安全的验证方式； 
  * 只勾选V2签名7.0以下机型会在直接安装完后显示未安装，7.0及以上机型使用V2的方式验证成功安装； 
  * 同时勾选V1和V2对所有机型成功安装（建议使用）.

* 方式二 Gradle 文件中修改
```
signingConfigs {  
    debug {  
        v1SigningEnabled true  
        v2SigningEnabled true  
    }  
    release {  
        v1SigningEnabled true  
        v2SigningEnabled true  
    }  
} 
```

---

**知识来源：**
[Android开发之签名V1和V2的区别](https://blog.csdn.net/francisbingo/article/details/78655848)
[Android 签名时 v2 与 v1 的选择](https://blog.csdn.net/lonewolf521125/article/details/74535413)
