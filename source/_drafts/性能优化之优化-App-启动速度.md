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

    App 所在进程存活，并且所有 Activity 均未被销毁，热启动做的只是把应用从后台切换至前台，如果 Activity 实例中的被回收的实例变量会重新创建。

    具体场景为在微信中切换到桌面，后马上回到微信。

* 温启动

    Activity 被杀死但是 App 所在的进程在内存中依然存在。

#### 启动时间组成

很多时候解决问题需要我们拥有上帝视角，全面把控事态的发展，跳出事件本身，以俯视的角度看待整个流程，找出问题的关键点。总的来说我们需要纵观全局来看待问题、解决问题。

从在 Launcher 中点击 App 的 icon 到 App 的第一面展示的时间差称为冷启动时间，那么这段时间整个系统都经历了什么。用上帝之眼看待整个流程，从而寻找解决问题的切入点，首先我们分析一下整个过程都经历了什么。

在 Launcher 中我们点击 App 图标，经过一系列过程，涉及 AMS、 Binder、IPC 等组件和过程。

在启动 App 的主页面之前，系统操作如下：
    
1. 创建并启动 App 所在应用
2. 启动 App 后会马上展示系统的 start window
3. 创建 App 进程

在系统创建完成 App 所在的进程后，App 进程负责完成下列一系列步骤：

1. 创建 App 对象
2. 启动 App 主线程
3. 创建主 Activity
4. 加载 View 视图
5. 布局屏幕
6. 执行初始化绘制 View

在 App 进程完成第一帧的绘制后，start window 会被替换，此时在屏幕上展示的就是在主 Activity 加载的布局。

![App 页面替换](/../images/2019_06_01_01.png)

根据 [Colt McAnlis](https://www.youtube.com/watch?v=Vw1G1s73DsY) 在 Yutobe 上的关于 App 启动时间的描述，以上过程可以形象的图形化表示如下：

![App 启动时间](/../images/2019_06_01_02.png)



#### 为什么是白屏或黑屏


官方图

拆解为两部分。

分析、找出优化点

#### 启动时间统计



#### 解决方案


更换主题

截图 + 解决方法 + 截图

延时加载


CPU profile 
trace 文件
命令行
Displayed Time

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

[Youtube 性能优化视频](https://www.youtube.com/watch?v=Vw1G1s73DsY)


https://www.zhihu.com/question/35487841