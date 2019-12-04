---
title: '小程序学习笔记(三):Js 文件代码结构和 Page 页面的生命周期'
tags:
---

### Js 文件代码结构

在微信小程序中， js 文件默认代码包含了我们可能用到的代码结构，整个页面执行了一个 Page({..}) 方法，参数是一个 Object 类型的对象，用来指定页面的初始数据(data)、生命周期函数(on开头的函数)、事件处理函数等。

### 生命周期函数

MINA 提供了 5 个生命周期函数：

* onLoad:页面加载，一个页面只会调用一次
* onShow：页面显示，每次打开都会调用
* onReady：监听页面完成初次渲染，一个页面只会调用一次
* onHide：监听页面隐藏
* onUnload：监听页面卸载

页面生命周期图解：

![页面生命周期图解](https://res.wx.qq.com/wxdoc/dist/assets/img/page-lifecycle.2e646c86.png)


### 数据绑定

在小程序中数据绑定有两种方式：

* 初始化数据的数据绑定

将这些数据直接写在 Page 方法参数的 data 对象下面。



> 数据绑定在组件的属性中时，如 image 的 src 属性，则要添加双引号，如 `<image src="{{avatar}}" />`；在内容型数据绑定，则不需要添加双引号，如 `<text >{{readingNum}}</text>`。

使用页面生命周期图解理解初始化数据绑定的过程：

1. 当页面执行 onShow 函数后，逻辑层收到一个通知(Notify)。
2. 逻辑层将 data 对象以 json 的形式发送端  View 视图层。
3. 视图层接收到数据化数据后，开始渲染并显示初始化数据(First Render)，最终将数据显示在屏幕上。

**复杂对象的调用**

js 文件：
```
data: {
        object:{
            date: "19.12.04",
        },
        collectionNum:{
            array:[109]
        },
    },
```
wxml 文件：

```
<text >{{collectionNum.array[0]}}</text>
<text >{{object.date}}</text>
```


* 使用 setData 方法来做数据绑定

可以将这种数据绑定的方式理解为数据更新，这样的数据更新中引起页面的重新渲染(Rerender)。

```
    onLoad:function(){
        // 会更高 data 中相应字段的值
        this.setData({
            title:"OnLoad Change Content",
            "collectionNum.array[0]":111
        })
    },
```