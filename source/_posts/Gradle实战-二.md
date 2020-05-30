---
title: Gadle实战(二)：声明 Task
date: 2020-01-16 19:02:23
tags: [Gradle 基本原理,Gradle in action]
---

## 1. Gradle API 中的 Task 

在默认情况下，每个创建的 Task 为 `org.gradle.api.DefaultTask` 类型的，为 `org.gradle.api.Task` 的实现类。

## 2. 声明 Task 的动作

动作（action）就是 **在 task 中合适的地方放置构建逻辑**。

Task 提供了两个相关的方法来声明动作：`doFirst(Closure)`、`doLast(Closure)`，当 Task 被执行
时，动作逻辑被定义为的闭包参数，被依次执行。

```
task helle{
    // 具体动作被封装在闭包中
    doLast{
        println 'last'
    }
}
```
<!-- more -->

以下展示如何声明 Task 动作。

### 2.1 声明 task 动作

```
task helle{
    doLast{
        println 'last'
    }
    doFirst{
        println 'First'
    }
}
```
### 2.2 为已有的 task 添加动作

为已有的 task 添加动作，可以实现对不是自己编写的 Task **填加自己的逻辑**，这在项目开发中是十分重要的。

为 Java 插件的 `compileJava` 添加一个 doFirst 动作：

```
compileJava.doFirst{
    println "add action"
}
```

compileJava 为 Java 插件实现的一个 Task，可以为该 Task 添加自己的动作，执行相应的逻辑。

## 3. 访问 DefaultTask 属性

Task 有两个属性：

* group(定义 Task 的逻辑分组) 
* description（描述Task 的作用)

在定义 Task 时可以指定这两个属性，为 Task 提供相应标识。

```
task printVersion(group:'version',description:'print project version'){
    // 两者等效，因为可以对 project 对象进行隐式调用
    logger.quiet "Verison is $version"
    logger.quiet "Verison is ${project.version}"
}
```

也可以通过 setter 方式对两者设置值：

```gradle
task printVersion{
    group = 'version'
    description = 'print project version'
    // 两者等效，因为可以对 project 对象进行隐式调用
    logger.quiet "Verison is $version"
    logger.quiet "Verison is ${project.version}"
}
```

当运行 gradle tasks 时，可以看到该 task 的分组和描述。

## 4. Task 依赖关系的建立

Task 的依赖关系通过两者方式建立：

1. 通过 `dependensOn` 方法,声明依赖一个或多个 task

```
task one << {
    println "one"
}
task two << {
    println "two"
}
// 声明多个依赖
task three(dependsOn:[two,one]) {
    println "three"
}
task four {
    println "three"
}
four.dependsOn('three')
```

2. 在 Task 调用其他 Task 相关内容

```
task test{
    dest = one.output.file
}
```
则依赖关系为： test 依赖 one。

task 依赖的执行顺序：**被依赖项先于依赖项先执行**。

## 5. Task 终结器

有时候需要在 Task或者依赖的 Task 执行完毕后，需要清理资源，Gradle 提供了 终结器 Task。


Task 终结器即使在终接器失败了，Task 也会按照预期执行。

```
task first{
    doFirst{
        sout 'first'
    }
}
task second{
     doFirst{
         sout 'second'
     }
}
first.finalizedBy second
```

执行 first 会自动触发 second：

> ./gradlew first

```
first
second
```


## 6. 可以在 Gradle 构建脚本中添加 Groovy 代码


build.gradle 文件：
```
...
version = new ProjectVersion(0,1)
class ProjectVersion{
    Integer major
    Integer manor

    ProjectVersion(Integer major,Integer manor){
        this.major = major
        this.minor = minor
    }
}
...
```
## 7. Task 配置块

当定义的 Task 不存在动作（action）时，则称为 **Task 配置**，比如：

```
task test{
    println 'test'
}
```

但是需要注意的一点是：**Task 配置永远在 Task 动作执行之前执行**,这牵涉到 Gradle 的生命周期阶段：初始化、配置、执行阶段，具体见下文。



## 8. Gradle 构建生命周期阶段

无论何时执行 Gradle 构建，都会运作三个不同的生命周期阶段：**初始化、配置、执行**，其其执行顺序如下：

![](Gradle实战-二/2020_01_17_01.png)

* 初始化阶段

Gradle 为 *每一个项目创建了一个 Project 实例*，此时分为单模块项目和多模块项目。这个阶段，**所有的构建脚本都不会执行**。


* 配置阶段

Gradle 构造一个模型来表示任务(Task)，并且参加到项目构建过程中。Gradle 采用了 **增量式构建** 特性，决定了模型中的 Task 是否会被执行，不会被执行的 Task 被标记为 UP-TO-DATE 。

**项目每一次构建时，任何 Task 配置块都可以被执行，即使只执行 gradle tasks。**


* 执行阶段

执行 Gradle 命令执行，这一阶段 Task 会按照正确的顺序执行，其顺序有 Task 之间的依赖关系决定的， taskA 依赖与 taskB，那么执行顺序为 taskB -> taskA 。

## 9. 通过声明 Task 的 inputs 和 outputs 属性的方式，引入增量式构建特性

增量式更新在 Gradle 中十分重要，该特性充分提高构建性能。

Gradle 通过比较两个构建的 Task 的 `inputs` 和 `outputs` 是否变化，来判断这个 Task 是否是最新的。如果自最后一个 Task 执行以来，如果 `inputs` 和 `outputs` 没有发生变化，那么认为该 Task 为最新的，则不会执行；反之则该 Task 会执行。

inputs 可以为一个目录、一个或者多个文件、一个或者多个属性，一个 Task 的 output 可以是一个目录或者任意个文件，对应于 Gradle API，inputs 和 outputs 被定义 DefaultTask 的属性，其对应类分别为 TaskInputs、TaskOutputs。

```
task printVersion{
    inputs.property('release',version.release)
    outputs.file versionFile

    doLast{
        version.release = true
        // 通过 ant 修改文件属性值
        ant.propertyfile(file:versionFile){
            entry(key:'release',type:'string',operation:'=',value:'true')
        }
    }
}
```

在一系列 Task 执行过程中，如果 printVersion 任务的 inputs、outputs 对应的属性没有发生变化，那么在任意 Task 执行链中就不会执行该 Task，不被执行的 Task 会在输出日志中被标记 `UP-TO-DATE` 的标记。

Task 的 inputs 和 outputs 是在配置阶段执行的，用来连接 Task 依赖的，所以需要在 Task 配置块中定义。