---
title: Flutter核心技术与实战(六)：动画实现
tags:
---



## 动画实现的方式


### Animation、AnimationController 与 Listener


为了实现动画，在 Flutter 中需要 Animation、AnimationController 与 Listener 的配合使用：

* Animation

Flutter 实现动画的核心类，根据预定规则，在单位时间内持续输出动画的当前状态。Animation 仅仅用了提供动画数据，不负责动画渲染。

* AnimationController


AnimationController 用于管理 Animation，用来设置动画的时长、启动动画、暂停动画等。

* Listener

Listener 是 Animation 的回调函数，用来监听动画的进度变化，需要在这个函数中，根据动画的当前值重新渲染组件，实现动画的渲染。

下面是一个示例：

```
class _AnimateAppState extends State<AnimateApp> with SingleTickerProviderStateMixin {
  AnimationController controller;
  Animation<double> animation;
  @override
  void initState() {
    super.initState();
    //控制器：创建动画周期为1秒的AnimationController对象
    controller = AnimationController(
        vsync: this, duration: const Duration(milliseconds: 1000));
    //创建 Animation 对象L 创建从50到200线性变化的Animation对象
    animation = Tween(begin: 50.0, end: 200.0).animate(controller)
      // 添加监听
      ..addListener(() {
        setState(() {}); //刷新界面
      });
    //  通过控制器开启动画
    controller.forward(); 
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Center(
        child: Container(
        width: animation.value, // 将动画的值赋给widget的宽高
        height: animation.value,
        child: FlutterLogo()
      )));
  }
}

  @override
  void dispose() {
    controller.dispose(); // 释放资源
    super.dispose();
  }
```

因为 Animation 只是用于提供动画数据，不负责动画渲染，所以在 Widget 的build 方法中，获取动画的当前值，用于实现动画效果。同时在页面销毁时，要释放动画资源。


### AnimatedWidget 和 AnimateBuild 


在上面的实现方式中，Animation 仅提供当前动画的数据，不负责渲染，需要手动的调用 setState 来重构页面，而 Flutter 提供了两个类来完成类似的操作：AnimatedWidget 和 AnimateBuilder 。

两个类提供了两种不同的方式来简化以上方式，具体查看极客时间。


### 页面切换动画


实现功能的 Widget：Hero，通过 Hero，我们可以在两个页面的共享元素之间，做出流畅的页面切换效果。

具体查看极客时间。



