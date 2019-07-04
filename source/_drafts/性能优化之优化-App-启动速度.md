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

在清单文件中我们需要设置 Application 的 Theme，比如自己设置如下：

```
<application
        ...
        android:theme="@style/sixcatTheme">
```

        
我们需要查看 sixcatTheme 主题的继承关系

```
1. <style name="sixcatTheme" parent="Theme.AppCompat.Light.NoActionBar">

2. <style name="Theme.AppCompat.Light" parent="Base.Theme.AppCompat.Light"/>

3. <style name="Base.Theme.AppCompat.Light" parent="Base.V7.Theme.AppCompat.Light">

4. <style name="Base.V7.Theme.AppCompat.Light" parent="Platform.AppCompat.Light">

5. <style name="Platform.AppCompat.Light" parent="android:Theme.Holo.Light">

6. <style name="Theme.Holo.Light" parent="Theme.Light">


7. <style name="Theme.Light">
      <item name="windowBackground">@drawable/screen_background_selector_light</item>
   </style>

8. <style name="Theme">
```

层级自 1~8 自上而下具有继承关系。

可以看到在第 7 层中存在 windowBackground 属性。

看一下 windowBackground 的 drawable 显示如何：

![drawable](/../images/2019_06_11_01.jpg)

事实证明确实是这个属性决定着点击 App 后显示的白屏或黑屏，当然这也为解决此问题提供了一个思路。


官方图

拆解为两部分。

分析、找出优化点

### 启动时间统计

#### 方法一 使用 Logcat 查看启动时间

自 Android4.4 (API level 19) 以上的版本可以通过 AS 的 Logcat 查看 App 的启动时间。


Displayed 标识的时间包括 `启动相应的进程`以及 `绘制完成相应的 Activity`，主要过程包括过程如下：

* 启动相应进程
* 初始化相应对象
* 创建相应的 Activity 
* 加载布局
* 第一次绘制应用


![Displayed Time](/../images/2019_06_11_02.png)

要想看到 Displayed 属性，Logcat 不能设置任何过滤条件。

#### 方法二 使用 adb 查看启动时间

具体命令如下：

```
adb shell am start -W package/activity
```

具体事例

```
 adb shell am start -W com.sixcat/com.sixcat.views.activity.MainActivity
```
具体展示信息如下：

```
Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.sixcat/.views.activity.MainActivity }
Status: ok
Activity: com.sixcat/.views.activity.MainActivity
TotalTime: 2892
ThisTime: xxx(这个属性自己没有显示，但是有些资料上存在)
WaitTime: 2905
Complete
```
具体属性含义可以参见 [Android 中如何计算 App 的启动时间？](https://www.androidperformance.com/2015/12/31/How-to-calculation-android-app-lunch-time/),含义参见如下：

* WaitTime 就是总的耗时，包括前一个应用 Activity pause 的时间和新应用启动的时间；
* ThisTime 表示一连串启动 Activity 的最后一个 Activity 的启动耗时；
* TotalTime 表示新应用启动的耗时，包括新进程的启动和 Activity 的启动，但不包括前
一个应用 Activity pause 的耗时。

在日常开发中，我们一般只需要关心 TotalTime ，这个时间是自己应用的启动时间。


#### reportFullyDrawn()

如官方给出的示意图一样：

![image](https://developer.android.com/topic/performance/images/cold-launch.png)

如果应用的初始化过程中包括懒加载或者异步加载，上面的两种方法获得启动时间均不包含以上情况所消耗的时间。

如示意图所以 reportFullyDrawn() 可以自定义统计时间的结束为止，其展示时间为 apk 初始化到 reportFullyDrawn() 调用位置消耗的时间。

这样可以在初始化过程中的懒加载或异步加载后，调用 reportFullyDrawn()，从而统计启动时间。 

为了验证效果，在 MainActivity 的 initView() 方法做以下调整：

```
override fun initView(){
    ...
    Thread.sleep(3000L)
    reportFullyDrawn
    ...
}
```
添加这两行代码前后的 Displayed Time 分别如下：

```
2019-06-12 14:11:14.270 805-819/? I/ActivityManager: Displayed com.sixcat/.views.activity.MainActivity: +3s291ms

2019-06-12 14:14:59.536 805-819/? I/ActivityManager: Displayed com.sixcat/.views.activity.MainActivity: +6s151ms
```
两次日志的时间相差大致为 3s 。
Get It !!!



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

https://www.jianshu.com/p/4f10c9a10ac9