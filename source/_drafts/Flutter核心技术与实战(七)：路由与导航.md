---
title: Flutter核心技术与实战(六)
tags:
---


Flutter 中的路由导航，综合借鉴了 Android、IOS、React 的特点，简洁而不失强大。



## Flutter 中的路由管理


在 Flutter 中，页面跳转是通过 Route 和 Navigator 来管理的。

* Route 是页面的抽象，注意负责创建对应的页面，接收参数，响应 Navigator打开和关闭；
* Navigator 维护一个路由栈，用了管理 Route，Route 打开即入栈，Route 关闭即出栈，还可以直接替换栈内指定的 Route。

根据是否提前注册页面标识符，Flutter 中的路由管理可以分为两种方式：

* 基本路由

无需提前注册，在页面切换时需自己构建页面实例。

* 命名路由

需要提前注册页面标识符，在页面切换时，通过标识符直接打开新的路由。


## 基本路由

**要导航到一个新的页面**，需要创建一个 MaterialPageRoute 实例，调用 Navigator.push 方法，将新的页面压到堆栈的顶部。


MaterialPageRoute 其实是一种 **路由模板**，针对不同平台，实现路由创建及切换过渡动画。

**想要返回上一个页面**，则需要调用 Navigator.pop 方法，从堆栈中删除这个页面。


```

class FirstScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return RaisedButton(
      //打开页面
      onPressed: ()=> Navigator.push(context, MaterialPageRoute(builder: (context) => SecondScreen()));
    );
  }
}

class SecondPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return RaisedButton(
      // 回退页面
      onPressed: ()=> Navigator.pop(context)
    );
  }
}
```


## 命名路由

基本路由使用简单，适合页面不多的场景。

通过名字来指定页面的切换，必须先给应用程序 MaterialApp 提供一个页面名称映射规则，即路由表 routes，这样 Flutter 才知道名字和页面 Widget 的对应关系。


路由表实际上是一个 Map<String,WidgetBuilder>，key 为页面的名称，value为一个 WidgetBuilder 回调函数，在这个函数中创建对应的页面。定义完成后，可以使用 Navigator.pushName 来打开页面。


```

MaterialApp(
    ...
    //注册路由
    routes:{
      "second_page":(context)=>SecondPage(),
    },
);
//使用名字打开页面
Navigator.pushNamed(context,"second_page");
```


但是这样存在一个风险，如果我们打开一个不存在的页面怎么办（编码是将页面的 name 写错），这时 Flutter 提供了 UnKnownRoute 属性，对未知的路由标志进行统一处理，比如进入统一错误页面或者弹窗提示用户。


```

MaterialApp(
    ...
    //注册路由
    routes:{
      "second_page":(context)=>SecondPage(),
    },
    //错误路由处理，统一返回UnknownPage
    onUnknownRoute: (RouteSettings setting) => MaterialPageRoute(builder: (context) => UnknownPage()),
);

//使用错误名字打开页面
Navigator.pushNamed(context,"unknown_page");
```


## 页面参数

Flutter 提供了路由参数机制，可以在打开路由时传递相关参数，在目标页面通过 RouteSetting 来获取页面参数。

```

//打开页面时传递字符串参数
Navigator.of(context).pushNamed("second_page", arguments: "Hey");

class SecondPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    //取出路由参数
    String msg = ModalRoute.of(context).settings.arguments as String;
    return Text(msg);
  }
}
```

同时 Flutter 也提供了返回参数机制，在 push 目标页面是，可以设置目标页面关闭的监听函数，以获取返回参数，当目标页面可以在关闭路由时在监听函数中可以获得返回的参数。

```

class SecondPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: <Widget>[
          Text('Message from first screen: $msg'),
          RaisedButton(
            child: Text('back'),
            //页面关闭时传递参数
            onPressed: ()=> Navigator.pop(context,"Hi")
          )
        ]
      ));
  }
}

class _FirstPageState extends State<FirstPage> {
  String _msg='';
  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      body: Column(children: <Widget>[
        RaisedButton(
            child: Text('命名路由（参数&回调）'),
            //打开页面，并监听页面关闭时传递的参数
            onPressed: ()=> Navigator.pushNamed(context, "third_page",arguments: "Hey").then((msg)=>setState(()=>_msg=msg)),
        ),
        Text('Message from Second screen: $_msg'),

      ],),
    );
  }
}
```