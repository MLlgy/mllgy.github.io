---
title: Flutter核心技术与实战（五）：定制不同风格的 App 主题
tags:
---


## 定制主题

主题，又叫皮肤、配色。

所谓主题切换，只不过是在不同主题间更新使用到的资源（包括背景图片资源、字体颜色、字号大小等）以及配置集合而已。

视觉效果是易变的，将这些变化的部分抽离出来，将提供不同视觉效果的资源和配置按照主题进行分类，整合到一个统一的中间层去管理，这样可以实现主题的管理和切换。

在 Flutter 中，**通过 ThemeData 来统一管理主题的配置信息**。


通过 ThemeData 来自定义应用主题，可以实现 App 全局范围或者是 Widget 局部范围的样式切换。


## 全局统一的视觉风格定制

在 Flutter 中，应用程序类 MaterialApp 的初始化方法，为我们提供了设置主题的能力。我们可以通过参数 theme，选择改变 App 的主题色、字体等，设置界面在 MaterialApp 下的展示样式。

```

MaterialApp(
  title: 'Flutter Demo',//标题
  theme: ThemeData(//设置主题
      brightness: Brightness.dark,//明暗模式为暗色
      primaryColor: Colors.cyan,//主色调为青色
  ),
  home: MyHomePage(title: 'Flutter Demo Home Page'),
);
```

以上代码只是更改了主色调和明暗模式两个参数，但是主题下的按钮、文字颜色都会随之调整，在默认情况下，ThemeData 中的很多其他次级视觉属性都会受到主色调和明暗模式的影响。


## 局部独立的视觉风格定制

在 Flutter 中，**可以使用 Theme 来对 App 的主题进行局部覆盖**。

**Theme 是一个单子 Widget 容器**，通过设置其 data 属性，对其子 Widget 进行样式定制。


可以通过一下两种方式：

* 如果不想继承任何 App 全局的颜色或者样式

直接新建一个 THemeData 实例，依次设置对应样式。

* 如果不想在局部重写所有的样式，

则可以使用 copyWith 方法，继承 App 的主题，只更新部分样式。

```
// 新建主题
Theme(
    data: ThemeData(iconTheme: IconThemeData(color: Colors.red)),
    child: Icon(Icons.favorite)
);

// 继承主题
Theme(
    data: Theme.of(context).copyWith(iconTheme: IconThemeData(color: Colors.green)),
    child: Icon(Icons.feedback)
);
```


### 复用样式


除了自定义Material Design 规范中那些可自定义部分样式外，主题的另一个重要用途是样式复用。

比如为一段文字复用 Materia Design 规范中的 title 样式，可以通过 Theme.of(context) 方法，取出对应的属性，应用到这段文字的样式中。

Theme.of(context) 方法将向上查找 Widget 树，并返回 Widget 树中最近的主题 Theme。如果 Widget 的父 Widget 们有一个单独的主题定义，则使用该主题。如果不是，那就使用 App 全局主题。

```

Container(
    color: Theme.of(context).primaryColor,//容器背景色复用应用主题色
    child: Text(
      'Text with a background color',
      style: Theme.of(context).textTheme.title,//Text组件文本样式复用应用文本样式
    ));
```

## 识别平台

我们可以根据 `defaultTargetPlatform` 来判断当前应用所运行的平台



