---
title: APP 启动优化
tags:
---

主要面临的问题：
 APP 冷启动白屏。


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