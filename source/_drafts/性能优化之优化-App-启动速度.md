---
title: 性能优化之优化 App 启动速度
tags: [性能优化]
---


主要面临的问题：
 APP 冷启动白屏。

针对于这个问题，基本上主流的解决方法存在两个：

    1. 主题替换。
    2. 减少冷启动时主线程的工作量。

本文的重点不在解决问题方法，而在于解决问题的过程，包括影响启动速度的因素、时间统计工具、解决思路。


优化 App 启动速度主要是优化冷启动时的 App 启动速度，官方文档 ([App startup time](https://developer.android.com/topic/performance/vitals/launch-time)) 中指出优化冷启动方式的同时，温启动和热启动也会得到改善。

在分析过程之前，首先了解启动方式。

### 冷启动、温启动、热启动

* 冷启动
  
  App 安装后第一次启动或者杀死 App 进程后重新启动称为冷启动。冷启动的启动成本较高，一切都要重新建立，包括新建应用进程、初始化应用以及首页的布局、渲染等操作。


* 热启动



方法一

```
<item name="android:windowIsTranslucent">true</item>
```
在原来的 MainActivity 设定的主题中添加以上代码。

带来的效果：
透明效果，即点击 App icon 后的白屏时间变为 透明 状态，避免了白屏，但是两者的时间几乎相同。给人的错觉时点击 icon 后一段时间后，app 才调起，给人的印象不好。


方法二 主题替换



在 App 启动时


---- 

App 启动时间统计


```zsh
adb shell am start -W package/activity

```


[Google Developer](https://www.youtube.com/watch?v=Vw1G1s73DsY)