---
title: Kotlin协程学习(一):协程的基本了解
date: 2019-10-07 15:45:22
tags: [Kotlin 协程]
---

### 1. 协程是什么

* 其实是一套由 Kotlin 官方提供的 **线程 API**

可以理解为一个线程框架，它的好处是方便，最大的好处是 **可以在同一个代码块中进行多次线程切换**。

### 2. 协程的好处

更方便。

<!-- more -->
使用 Kotlin 协程可以用看起来同步的方式写出异步代码，这就是 Kotlin 著名的 **非阻塞式挂起**。

### 3. 使用 Kotlin 协程

* 基本功能：并发(多线程)

例如可以把任务切换到后台或前台执行

```
launch(Dispatchers.IO){
    doSomething(data)
}
launch(Dispatchers.Main){
    updateUI(data)
}
```


* 最大好处

Kotlin 最大的好处在于可以把：运行在不同线程的代码，写在同一个代码块中：

```
launch(Dispatchers.Main){// 开始：主线程
    val user = api.getUser()// 网络请求：后台线程
    name.text = user.name// 更新 UI：主线程
}
```

子线程和主线程使用 Kotlin 协程，可以消除了 Java 中的回调，避免回调地狱。



存在这么一个需要：需要合并两个独处接口的结果，然后操作。此时 Java 中传统的回调并不能很好的完成任务，此时很大可以会采用两次串行请求，然后合并进行操作，这样的两个独立的接口原本可以并行请求，现在只能串行请求，很明显网络的等待时间长了一倍，性能也差了一倍。而使用 Kotlin 协程则可以很好的解决这个问题：

```
launch(Dispatchers.Main){
    val avatar = async {api.getAvatar(user)}
    val logo = async { api.getLogo(user)}
    val merge = suspendingMerge(avatar,logo)
    show(merge)
}
```

协程消除了并发任务之间的协作难度，轻松写出复杂的并发代码。

这些才是协程的优势。
### 怎么用

* 简单

```
launch(Dispatchers.IO){
    doSomething(data)
}
```


launch 的具体含义是：创建一个新的协程，并在指定线程上运行它。

被创建、被运行的协程是什么？就是传给 launch 的代码，如 `doSomething(data)`，这一段连续代码称为协程。

当需要切换线程或者指定线程的时候，可以使用 协程。


### withContext

```
lauch(Dispatchers.IO){
    val image = api.getImage()
    launch(Dispatchers.Main){
        image.setImageBitmap(image) 
    }
}
```
但是仅仅使用 launch 并不能避免回调地狱，在协程中有一个函数： withContext()，这个函数可以指定线程来执行代码，并且在执行之后,**自动把线程切回来**，继续执行代码，使用 withContext 后，以上代码可以这样写：


```
launch(Dispatchers.Main){
    val image = withContext(Dispatchers.IO){
        api.getImage()
    }
    image.setImageBitmap(image)
}
```


协程消除了并发代码在协作时的嵌套，

```
launch(Dispatchers.IO){
    ...
    launch(Dispatchers.Main){
        ...
        launch(Dispatchers.IO){
            ...
        }

    }

}
```
使用 withContext 可以吧以上的嵌套关系直接写成上下关系的代码：

```
launch(Dispatchers.Main){
    withContext(Dispatchers.IO){
        ...
    }
    withContext(Dispatchers.IO){
        ...
    }
}
```

### suspend 函数


有 suspend 修饰符的函数，称为 suspend 函数，挂起函数。

代码执行到 suspend 函数的时候会被 suspend(挂起)，并且这个 "挂起" 是 非阻塞式的，它不会阻挡你的线程。



---

**知识来源：**


[【码上开学】学不会协程？很可能因为你看过的教程都是错的——Kotlin 的协程「用力瞥一眼」](https://www.bilibili.com/video/av67107689)