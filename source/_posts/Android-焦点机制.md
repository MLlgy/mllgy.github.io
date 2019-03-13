---
title: Android 焦点机制
date: 2019-03-11 11:08:29
tags: [焦点机制]
---


### No-touch Mode(非触摸模式) 和 Touch Mode(触摸模式)

Google 把触摸模式分为 No-touch Mode 和 Touch Mode。Android Phone 由于触摸屏，所以讨论的为 `触摸模式`，而像 Andorid TV、键盘、轨迹球一般为 `非触摸模式`。

### Focus(焦点)、Focusable(可聚焦)

在像 Android TV 、轨迹球这类设备上，我们可能需要通过遥控器等设备选择相应的选项，在交互过程中，我们选中的控件获得焦点，并通过颜色改变、高亮、突出等形式表现出来。
根据 Google 官方文档，在触摸模式下，其实没有没有焦点的概念，或者说此模式下的获得焦点的表现形式不同。在触摸模式下，Focus 以一种特别的方式 -- Focusable 存在。
<!-- More -->
根据用户的不同行为,两种模式可以不断切换：
在用户点击屏幕时，设备会进入触摸模式，而当用户点击轨迹球时，App 会立即退出触摸模式进入非触摸模式，并寻找一个控件获得焦点。

### 触摸模式与 Focusable

Focusable 此特殊模式是为接收文本输入的控件创建的，如 EditText。在触摸模式中，如果控件是可聚焦的(Focusable),只要用户点击该控件，该控件就会得到焦点，反之控件是不会获得焦点的。

Foucusable 其实为控件的一个属性，可以通过代码 `setFocusableInTouchMode` 或 xml中`android:focusableInTouchMode`设置控件是否可聚焦。

### setFocusableInTouchMode 和 setFocusable

* setFocusable:设置控件是否可以获得焦点，可以通过 isFocusable() 获得状态。
* setFocusableInTouchMode: 在触摸模式下，可以通过setFocusableInTouchMode 来设置控件是否可聚焦，可以通过isFocusableInTouchMode() 获得状态。

大部分控件的 `setFocusableInTouchMode` 属性均为 false ，至于 EditText 的 `setFocusableInTouchMode` 的属性为 true，这也就是为什么 EditText 会率先获得屏幕焦点的原因。

### 焦点监听与事件监听

当为控件设置可聚焦属性：
```
  <Button
        android:id="@+id/btnOne"
        android:layout_width="match_parent"
        android:text="one"
        android:focusableInTouchMode="true"   
        android:layout_height="wrap_content"/>
```
同时，为该控件设置了点击事件、焦点监听，此时是需要特别注意的：

```
btnOne.setOnclickListener{
    Log.e("TAG","click")
}
btnOne.setOnFocusChangeListener{ v, hasFocus ->
    Log.e("TAG","focus change" + hasFocus)
}
```

当我们点击 Button 时，此时 Log 日志：
```
TAG focus change true
```
而不会响应点击事件，想要 Button 响应点击事件，需要再次点击该按钮。
```
TAG click
```

在这种情况下，需要点击两次才能让 Button 响应点击事件：
1. Button 获得焦点
2. Button 响应点击事件

所以 Google 建议使用 `focusableInTouchMode` 之前，需要三思后行。

### descendantFocusability

`Defines the relationship between the ViewGroup and its descendants when looking for a View to take focus.`

该属性的字面意思: 子代获取焦点的能力。该属性定义的是当一个 子View 获取焦点时， ViewGroup 与 子View 之间的关系。

* **beforeDescendants**：Viewgroup 会优先其子类控件而获取到焦点
* **afterDescendants**：Viewgroup 只有当其子类控件不需要获取焦点时才获取焦点
* **blocksDescendants**：Viewgroup 会覆盖子类控件而直接获得焦点