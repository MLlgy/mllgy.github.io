---
title: Flutter核心技术与实战(六)
tags:
---

对于复杂的、视图层级较深的 UI 样式，一个属性可能跨越多层才能够传递给子组件，但是中间层使用不到这个属性也要接收该属性，过程冗余、繁杂。

对于数据的跨层传递，Flutter 提供了三种方案：InheritedWidget、Notification 和 EventBus。

## InheritedWidget

Inherited 在此处可以翻译为 内在的，意为向内部传递数据的 Widget。

InheritedWidget 是 Flutter 中的一个功能型 Widget，适用于在 Widget 树中共享数据的场景，可以实现在 Widget 树中跨层传递。

Theme 类是通过 InheritedWidget 实现的典型案例。

在子 Widget 中通过 Theme.of 方法找到上层 Theme 的 Widget，获取其属性的同时，**建立子 Widget 和父 Widget 的观察者关系**，当上层父 Widget 属性修改的时候，子 Widget 也会触发更新。

InheritedWidget 案例：计数器。

1. 自定义类CountContainer，继承 InheritedWidget。
2. 自定义属性 count，提供 of 方法，以便其子 Widget 能做在 Widget 树中找到它。
3. 重写 updateShouldNotify，在该方法中判断 InheritedWidget。 是否刷野重建，从而通知下层观察者组件更新数据时被调用到。


```

class CountContainer extends InheritedWidget {
  //方便其子Widget在Widget树中找到它
  static CountContainer of(BuildContext context) => context.inheritFromWidgetOfExactType(CountContainer) as CountContainer;
  
  final int count;

  CountContainer({
    Key key,
    @required this.count,
    @required Widget child,
  }): super(key: key, child: child);

  // 判断是否需要更新
  @override
  bool updateShouldNotify(CountContainer oldWidget) => count != oldWidget.count;
}
```

4. 使用 CountContainer 作为根节点，将 count 初始化为 0，在子吃Widget 中通过 CountContainer.of 方法找到 CountContainer ，获取计数状态 count 并展示。

```

class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
   //将CountContainer作为根节点，并使用0作为初始化count
    return CountContainer(
      count: 0,
      child: Counter()
    );
  }
}

class Counter extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    //获取InheritedWidget节点
    CountContainer state = CountContainer.of(context);
    return Scaffold(
      appBar: AppBar(title: Text("InheritedWidget demo")),
      body: Text(
        'You have pushed the button this many times: ${state.count}',
      ),
    );
}
```
这时子 Widget 就可以获得其父 Widget 的属性--count 了。


但是 InheritedWidget 只提供获取属性值的能力，不能对属性进行修改。如果想要修改，需要与 StatefulWidget 中的 State 配套使用。把在 InheritedWidget 中的修改数据的相关方法，全部移到 StatefulWidget 中的 State 中，而 InheritedWidget 只需要保持对他们的引用。

```

class CountContainer extends InheritedWidget {
  ...
  final _MyHomePageState model;//直接使用MyHomePage中的State获取数据
  final Function() increment;

  CountContainer({
    Key key,
    @required this.model,
    @required this.increment,
    @required Widget child,
  }): super(key: key, child: child);
  ...
}
```

在 State 中提供修改数据的方法，同样还是将 CountContaine 作为根节点，完成数据和修改方法的初始化。

```

class _MyHomePageState extends State<MyHomePage> {
  int count = 0;
  void _incrementCounter() => setState(() {count++;});//修改计数器

  @override
  Widget build(BuildContext context) {
    return CountContainer(
      model: this,//将自身作为model交给CountContainer
      increment: _incrementCounter,//提供修改数据的方法
      child:Counter()
    );
  }
}

class Counter extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    //获取InheritedWidget节点
    CountContainer state = CountContainer.of(context);
    return Scaffold(
      ...
      body: Text(
        'You have pushed the button this many times: ${state.model.count}', //关联数据读方法
      ),
      floatingActionButton: FloatingActionButton(onPressed: state.increment), //关联数据修改方法
    );
  }
}
```

因为 setState 会使 Widget 重建。

