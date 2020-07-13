---
title: RN 基本语法
tags:
---

## 块声明

使用 const 和 let 声明本地变量。const 声明的变量只能被一次赋值。



var 与 const、let 的不同

* var 作用域为函数
* const、let 作用域为块


## 箭头函数

=>

箭头函数内部的this总是指向定义时所在的对象


箭头函数用来声明匿名函数，与 function 的两点不同：

1. 关键字 this 在箭头函数的内部和外部含义相同，this 在function 在调用可以绑定其他对象；
2. 箭头函数在声明时没有定义参数，可以使用 如下方式进行传参：

```
(...args) => doSomething(args[0], args[1])
```
箭头函数不同情况下的声明：

```
x => Math.pow(x, 2)

(x, y) => Math.pow(x, y)

```

函数体没有使用大括号，则作为一个表达式。

## 解构

从一个对象或数组中提取多个键，并将值分配给局部变量。

## import/export

每个文件默认有一个导出，并且可以导入该导出的值而无需按名称引用。其他所有进出口都必须命名。

## 默认参数

当传入的参数为 undefined 时，则该参数值被设置为默认参数。

## 类

RN 存在类的概念，可以通过 extends 实现继承。

## 动态属性

ES6 允许创建动态属性名的属性。

```
// dynamic property key name
var type = 'name';
// set dynamic property between square brackets
var o = {
	id: 1131,
	gender: "Male",
	[type]: "Sample Name"
}
// { id: 1131, gender: "Male", name: "Sample Name" }
console.log(o);

// 可以将表达式放在方括号中
var o1 = {
	[type + '_' + i++]: "One",
	[type + '_' + i++]: "Two",
	[type + '_' + i++]: "Three",
}

// { name_1000: "One", name_1001: "Two", name_1002: "Three" }
console.log(o1);
```


## ES2016+

ES6 引入更多新特性，通过 Babel 可以更加方便的使用这些特性。

### 静态属性

static

## 属性初始化

在 ES6 之前，为 function 绑定实例对象需要在 constructor 进入如下操作：

```
constructor(){
    this.func = this.func.bind(this);
}
```

在 ES6 可以进行如下操作：


```
func = () => ...
```

因为箭头函数内部的this总是指向定义时所在的对象。


### 展开语法

[展开语法](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Spread_syntax)



```
function sum(x, y, z) {
  return x + y + z;
}

const numbers = [1, 2, 3];

console.log(sum(...numbers));
// expected output: 6

console.log(sum.apply(null, numbers));
// expected output: 6
```

## Async、Await

在 function 前使用关键字 async ，返回 Promise 对象，在 async 方法中使用 await 关键字等待返回结果。

```
const asyncReadFile = async function () {
  const f1 = await readFile('/etc/fstab');
  const f2 = await readFile('/etc/shells');
  console.log(f1.toString());
  console.log(f2.toString());
};
```

## JSX

JSX 扩展了 JS，引入了一种新的表达式，可以在任何表达式使用的地方使用 JSX。

JSX是使用React.createElement()API 的快捷方式，它更加简洁，易于阅读。

JSX和XML一样都是基于标签的。每个标签<View />都将转换为对的调用React.createElement()。

任何属性都将成为props实例化组件的属性。属性可以是类似的字符串foo='hello'，或者当用大括号括起来时bar={baz}（可以引用变量baz），可以将它们插入JavaScript表达式。

标签可以是自闭标签，例如<View />，也可以同时包含开始标签和结束标签，例如<View></View>。要包含子元素，您将需要使用一个开始和结束标签，并将子标签放入其中。

**菜鸟篇**

在 React 中推荐使用 JSX 来描述界面。

元素是构成 React 应用的最小单位，JSX 就是用来声明 React 当中的元素。

与浏览器的 DOM 元素不同，React 当中的元素事实上是普通的对象，React DOM 可以确保 浏览器 DOM 的数据内容与 React 元素保持一致。

要将 React 元素渲染到根 DOM 节点中，我们通过把它们都传递给 ReactDOM.render() 的方法来将其渲染到页面上：

```
<body>
	<div id="root"></div>
	<script type="text/babel">
	const element = <h1 className="foo">Hello, world</h1>;
	ReactDOM.render(element, document.getElementById('root'));
	</script>
</body>
```


可以在 JSX 中使用表达式，表达式写在 {} 中。


# React 组件

组件是所有 UI 的基本组成单元。

React Native 组件由 Natvie 组件进行真正渲染，而 RN 需要管理这种映射关系。

```
// 使用函数定义一个组件
function HelloMessage(props) {
    return <h1>Hello World!</h1>;
}

// 也可以使用 ES6 的 class 来定义个一个组件
class Welcome extends React.Component {
  render() {
      // 使用 this.props 对象接收参数
    return <h1>Hello {props.name}!</h1>;
  }
}

// 为用户自定义一个组件，可以向组件传递参数
const element = <HelloMessage name="Tom"/>;
 
ReactDOM.render(
    element,
    document.getElementById('example')
);
```

# React  State(状态)

React 里，只需更新组件的 state，然后根据新的 state 重新渲染用户界面（不要操作 DOM）。

```
class Clock extends React.Component {
  constructor(props) {
    super(props);
    this.state = {date: new Date()};
    // 另一种写法
    this.setState({
      date: new Date()
    });
  }
 
  render() {
    return (
      <div>
        <h1>Hello, world!</h1>
        // 获得最新的 data 值
        <h2>现在是 {this.state.date.toLocaleTimeString()}.</h2>
      </div>
    );
  }
}
 
ReactDOM.render(
  <Clock />,
  document.getElementById('example')
);
```


任何状态始终由 **某些特定组件所有**，并且从该状态导出的任何数据或 UI **只能影响树中下方的组件**，其意思是父组件可以将自己的 state 值传给嵌套组件，反之不可以。


## 组件 API

setState


# 生命周期 API

组件具有生命周期：它们被实例化，安装，渲染，并最终被更新，卸载和销毁。

生命周期的各个阶段：

## 安装周期

* constructor(object props)
组件类被实例化。构造函数的参数props是父元素指定的元素的initial 。您可以通过将对象分配给来为元素指定初始状态this.state。目前，尚未为该元素呈现任何本机UI。

* componentWillMount()
在第一次进行渲染之前，仅调用一次此方法。此时，仍没有为此元素呈现本机UI。

* render() -> React Element
render方法必须返回一个React Element进行渲染（或为null，则不渲染任何内容）。

* componentDidMount()
第一次进行渲染后，仅调用一次此方法。至此，此元素的本机UI已完成渲染，可以直接访问以this.refs进行直接操作。如果您需要使用进行API异步调用或执行延迟的代码setTimeout，通常应使用此方法。

## 更新周期

* componentWillReceiveProps(object nextProps)
此组件的父级传递了一组新的props。该组件将重新渲染。在调用方法this.setState()之前，您可以选择调用以更新此组件的内部状态render。

* shouldComponentUpdate(object nextProps, object nextState) -> boolean
根据下一个值props和state，组件可以决定重新呈现不重新渲染。此方法的基类实现始终返回true（组件应重新呈现）。为了优化，覆盖此方法，并检查是否任一props或state已被修饰，例如运行在这些对象中的每个键/值的相等测试。返回false将防止render调用该方法。

* componentWillUpdate(object nextProps, object nextState)
在决定重新渲染之后，将调用此方法。您可能无法在此处调用 this.setState()，因为更新正在进行中。

* render() -> React Element
假定shouldComponentUpdate返为 true，则调用此方法。render方法必须返回一个React Element进行渲染（或为null，则不渲染任何内容）。

* componentDidUpdate(object prevProps, object prevState)
重新渲染发生后，将调用此方法。此时，此组件的本机UI已更新，对应 render 返回的 React Element 。


# 核心组件


# 数据管理

数据管理的常用选项：

* Component State

将数据存储在组件中的 state 中，是最简单的数据管理方式。

* Redux

第三方数据管理库，Redux提供了一个维护应用程序状态的对象 store，并可以在状态更改时通知您的React组件。


## Component State

构建组件的最佳方法通常是将它们分为两类：容器和组件。

* 容器

容器组件是一些和应用逻辑相关的组件，简称为容器。

* 显示组件


绝大多的组件不应包含业务逻辑，仅仅作为显示，它们唯一的输入为 prop 数据，被简称为组件。



## Redux 

（1）Web 应用是一个状态机，视图与状态是一一对应的。

（2）所有的状态，保存在一个对象里面。



### 基本 API

* Store

Store 就是保存数据的地方，你可以把它看成一个容器。整个应用只能有一个 Store。

Redux 提供createStore这个函数，用来生成 Store。

```
import { createStore } from 'redux';
const store = createStore(fn);
```
createStore函数接受另一个函数作为参数，返回新生成的 Store 对象，从后文可以看到这个函数就是  reducer。


* State


如果想得到 **某个时点的数据**，就要对 Store 生成快照。这种时点的数据集合，就叫做 State。
当前时刻的 State，可以通过store.getState()拿到。

```
import { createStore } from 'redux';
const store = createStore(fn);

const state = store.getState();
```

Redux 规定， 一个 State 对应一个 View。只要 State 相同，View 就相同。

* Action

State 的变化必须是 View 导致的。**Action 就是 View 发出的通知，表示 State 应该要发生变化了**。Action 是一个对象。其中的type属性是必须的，表示 Action 的名称。

```
const action = {
  type: 'ADD_TODO',
  payload: 'Learn Redux'
};
```

Action 的名称是ADD_TODO，它携带的信息是字符串Learn Redux。

可以这样理解，Action 描述当前发生的事情。改变 State 的唯一办法，就是使用 Action。它会运送数据到 Store。

* Action Creator

View 要发送多少种消息，就会有多少种 Action，麻烦，定义一个名为 Action Creator 函数生成 Action。


```
const ADD_TODO = '添加 TODO';

function addTodo(text) {
  return {
    type: ADD_TODO,
    text
  }
}

const action = addTodo('Learn Redux');
```

* store.dispatch()

store.dispatch()是 View 发出 Action 的唯一方法。store.dispatch接受一个 Action 对象作为参数，将它发送出去。

```
store.dispatch(addTodo('Learn Redux'));
```


* Reducer 

Store 收到 Action 以后，必须给出一个新的 State，这样 View 才会发生变化。这种 State 的计算过程就叫做 Reducer。

Reducer 是一个函数，它接受 Action 和当前 State 作为参数，返回一个新的 State。

```
const defaultState = 0；
const reducer = function (state = defaultState, action) {
    switch(action.type){
        case 'ADD':
         return state + action.payload;
        default:
            return state;

    }
  return new_state;
};

const state = reducer(1, {
  type: 'ADD',
  payload: 2
});
```

state.dispatch 方法会触发 Reducer 的自动执行，所以在创建 store 时需要传入 reducer，具体如下：

```
import { createStore } from 'redux';
const store = createStore(reducer);
```

* store.subscribe()

Store 允许使用store.subscribe方法设置监听函数，一旦 State 发生变化，就自动执行这个函数。store.subscribe方法返回一个函数，调用这个函数就可以解除监听。


```
import { createStore } from 'redux';
const store = createStore(reducer);

let unsubscribe = store.subscribe(listener);
// 解除监听
unsubscribe();
```

对于 React 项目。只要把 View 的更新函数(render()、setState) 放入 listener 中，就可以实现 View 的自动渲染。


---



[](http://www.ruanyifeng.com/blog/2016/09/redux_tutorial_part_one_basic_usages.html)


[Web 开发技术](https://developer.mozilla.org/zh-CN/docs/Web/Tutorials)


[React Navigation中文网](https://www.reactnavigation.org.cn/docs/guide-intro)