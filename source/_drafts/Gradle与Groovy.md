---
title: Gradle与Groovy
tags:
---




DSL：Domain Specific Language 领域特定语言



----



## Gradle 脚本的执行时序

Gradle 脚本的执行分为以下三个过程：

* 初始化

分析项目有哪些 module 将被构建，为每个 module 创建对应的 Project 实例，这个阶段项目的 `setting.gradle` 文件会被解析。


* 配置

处理所有模块的 build 脚本，处理依赖，属性等。

这个时候每个模块的 `build.gradle` 文件都会被解析并配置，这时候会构建整个 task 链表（存在依赖关系的 task 的集合）。


* 执行

执行 task 链表中的某个 task，这个 task 所依赖的 task **会被提前执行**。


执行以下 task，Gradle执行的时候遵循如下顺序”：

执行命令：./gradlew clean


执行顺序：

1. 首先解析 `settings.gradle` 来获取模块信息，这是初始化阶段；
2. 然后配置每个模块，**配置的时候并不会执行 task**；
3. 配置完了以后，如果存在回调 `project.afterEvaluate`，它表示所有的模块都已经配置完了，可以准备执行task了；
4. 执行指定的task。


## Task

Task  为 Gradle 的执行单元，Gradle 通过一个个 Task 去执行具体的构建任务。

定义 Task

```
task customTask{
    println 'customTask'
}
```

执行自定义 Task

`./gradlew customTask`

执行结果：

```
customTask
```


在执行命令道：`./gradlew clean` 时也会打印相同的结果。

通过上述方案定义 Task，括号内的代码会在配置阶段执行，这也就是说，只有我们执行任意一个 Task，自定义的代码段都会执行，因为**每个 Task 执行之前都需要进行一次完整的配置。**

不过很多时候我们不需要写配置代码，想要括号中的代码仅仅在执行指定 Task 的时候执行，这个时候可以通过 doLast 或者 doFirst 来完成。

* doFirst

在指定 task 执行前执行的操作。

* doLast

在指定 Task 执行后执行的操作。


具体示例如下：


```
task customTask{
    println 'customTask'
}

customTask.doLast{
    println "customTask Last"
}

customTask.doFirst{
    println "customTask First"
}
```


运行定义的 Task：


```
$:app gradle customTask
```
以下是打印结果：
```
// Gradle 脚本的配置阶段
> Configure project :
customTask
// 执行 Gradle 脚本的指定 Task
> Task :customTask
customTask First
customTask Last

BUILD SUCCESSFUL in 0s
1 actionable task: 1 executed
```

可以看到的是在 配置执行阶段只会执行 customTask 中的代码，并不会执行 doFirst、doLast 代码块中的代码段，

---


任玉刚 Gradle 系列：

[Gradle从入门到实战 - Groovy基础](https://blog.csdn.net/singwhatiwanna/article/details/76084580)
[全面理解Gradle - 执行时序](https://blog.csdn.net/singwhatiwanna/article/details/78797506)

[全面理解Gradle - 定义Task](https://blog.csdn.net/singwhatiwanna/article/details/78898113)