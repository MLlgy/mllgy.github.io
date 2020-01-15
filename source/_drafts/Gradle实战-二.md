---
title: Gadle实战(一)：使用 Task
tags:
---


## 使用 Task

在默认情况下，每个创建的 Task 为 org.gradle.api.DefaultTask 类型的，为 org.gradle.api.Task 的实现类。

### 声明 Task 的动作


动作（action）就是在 task 中和合适的地方放置构建逻辑，Task 提供了两个相关的方法来声明动作：doFirst、doLast。


声明一个包含动作的 task

```
task helle{
    doLast{
        println 'last'
    }

    doFirst{
        println 'First'
    }
}

hello.doLast{
    println 'custom last'
}
```


可以为已有的 task 添加一些动作，这对不是自己编写的 task 执行自定义的逻辑是十分有用的。

为 Java 插件的 compileJava 添加一个 doFirst 动作：
```
compileJava.doFirst{
    sout "xx"
}
```

### f访问 DefaultTask 属性

使用第三方库 logger ：

```
task printVerison.doLast{
    logger.quiet "Verison is $version"
}
```

Task 有两个属性：group(定义 Task 的逻辑分组) 和 description（描述Task 的作用)