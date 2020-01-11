---
title: Flutter核心技术与实战（五）：定制不同风格的 App 主题
tags:
---

## 手势操作

手势操作分为两类：

* 原始的指针事件

表示用户在屏幕上触摸行为触发的位移行为。

* 手势识别

表示多个原始指针事件的组合操作，如点击、双击等，是指针事件的语义化封装。


## 指针事件


指针事件表示用户交互的原始触摸数据：

* 手指接触屏幕：PointerDownEvent
* 手指在屏幕上移动：PointerMoveEvent
* 手指抬起：PointerUpEvent
* 触摸取消：PointerCancelEvent

手指触摸屏幕，产生触摸事件，Flutter 会确定手指与屏幕接触的位置上究竟有哪些组件，并将触摸事件交给 **最内层** 的组件去响应。

与浏览器中的事件冒泡机制类似，事件从最内层开始，**沿着组件树向上冒泡分发**，但是不能想浏览器中一样取消或者停止事件的分发，只能通过 hitTestBehavior 去调整在测试期如何表现，比如将触摸事件交给子组件，或者交给视图层之下的组件去响应。


Flutter 中提供了 Listener Widget，**可以监听子 Widget 的原始指针事件**.
```

Listener(
  child: Container(
    color: Colors.red,//背景色红色
    width: 300,
    height: 300,
  ),
  onPointerDown: (event) => print("down $event"),//手势按下回调
  onPointerMove:  (event) => print("move $event"),//手势移动回调
  onPointerUp:  (event) => print("up $event"),//手势抬起回调
);
```

## 手势识别


从组件层面监听手势，Flutter 提供了 GestureDetector，GestureDetector 是一个处理各种高级用户触摸行为的 Widget，与 Listener 一样，也是一个  **功能性组件**。


使用 GestureDetector 的示例：

```

//红色container坐标
double _top = 0.0;
double _left = 0.0;
//使用Stack组件去叠加视图，便于直接控制视图坐标
Stack(
  children: <Widget>[
    Positioned(
      top: _top,
      left: _left,
      child: GestureDetector(//手势识别
        child: Container(color: Colors.red,width: 50,height: 50),//红色子视图
        onTap: ()=>print("Tap"),//点击回调
        onDoubleTap: ()=>print("Double Tap"),//双击回调
        onLongPress: ()=>print("Long Press"),//长按回调
        onPanUpdate: (e) {//拖动回调
          setState(() {
            //更新位置
            _left += e.delta.dx;
            _top += e.delta.dy;
          });
        },
      ),
    )
  ],
);
```
虽然在 GestureDetector 中对 Widget 同时监听了多个手势事件，但是最终只有一个手势能够得到本次事件的处理权。

对于多个手势的识别，Flutter 引入了 **手势竞技场（Arena）**，用来识别哪个手势可以响应用户事件。手势竞技场会考虑用户触摸屏幕的时间、位移、拖动方向，来确定最终手势。


## 手势竞技场的具体实现



GestureDetector 内部对每一个手势都建立了一个工厂类，而工厂类的内部会使用手势识别类（GestureDetector），来确定当前处理的手势。

所有的手势工厂了都会交给 RawGestureDetector 类，以完成监测手势的大量工作：使用 Listener 监听原始指针事件，并在状态改变时把信息同步给所以的手势识别器（GestureDetector），然后这些手势会在竞技场决定最后有谁响应用户事件。


在父子关系的视图中，手势竞技场会同时检查父、子视图的手势，通常会确认由子视图来响应事件。


```

GestureDetector(
  onTap: () => print('Parent tapped'),//父视图的点击回调
  child: Container(
    color: Colors.pinkAccent,
    child: Center(
      child: GestureDetector(
        onTap: () => print('Child tapped'),//子视图的点击回调
        child: Container(
          color: Colors.blueAccent,
          width: 200.0,
          height: 200.0,
        ),
      ),
    ),
  ),
);
```

只有子视图响应了用户的点击事件。



为了让父视图识别到手势，需要同时使用 RawGestureDetector 和 GestureFactory，来改变竞技场决定有谁来响应用户事件的结果。

需要自定义一个手势识别器，让这个识别器在竞技场被 PK 失败时，能够再把自己重新添加回来，以便接下来还能继续去响应用户事件

需要使用的相关类RawGestureDetector（原始事件识别器）、xxxGestureRecognizer（手势识别器）、GestureRecognizerFactoryWithHandlers（工厂类）.

自己构建手势识别器：

```

class MultipleTapGestureRecognizer extends TapGestureRecognizer {
  @override
  void rejectGesture(int pointer) {
    acceptGesture(pointer);
  }
}
```


```

RawGestureDetector(//自己构造父Widget的手势识别映射关系
  gestures: {
    //建立多手势识别器与手势识别工厂类的映射关系，从而返回可以响应该手势的recognizer
    MultipleTapGestureRecognizer: GestureRecognizerFactoryWithHandlers<
        MultipleTapGestureRecognizer>(
          () => MultipleTapGestureRecognizer(),
          (MultipleTapGestureRecognizer instance) {
        instance.onTap = () => print('parent tapped ');//点击回调
      },
    )
  },
  child: Container(
    color: Colors.pinkAccent,
    child: Center(
      child: GestureDetector(//子视图可以继续使用GestureDetector
        onTap: () => print('Child tapped'),
        child: Container(...),
      ),
    ),
  ),
);
```