---
title: Gadle实战(一)：
tags:
---


> 最基础的Gradle构建块-project和task，以及它们是如何映射到 Gradle 的 API 上面的。
1. 声明简单的 task
2. 编写自定义的 task 类
3. 如何获取 task 属性
4. 定义显示和隐式的 task 依赖
5. 添加增量式构建支持
6. 使用 Gradle 的内置 task 类型
7. Gradle 的构建生命周期
8. 编写使用闭包和监听器 实现的生命周期钩子

## 构建块

包括 project、task、propert。


Gradle 使用领域驱动设计（DDD）的原理为自己的领域构建软件模型，因此 Gradle API 中有相应的类来表示 project和task。


### project


在 Gradle 中，一个 project 代表一个正在构建的组件，或者一个想要完成的目标：比如部署应用程序。

每个 Gradle 构建脚本至少定义一个项目。

**当构建进程启动后，Gradle 基于 build.gradle 中的配置实例化 org.gradle.api.Project 类，并且能够通过 project 变量使其隐式使用**。


### Task




### 属性

每个 Project 和 Task 实例都提供了很多可以通过 setter 和 getter 方法访问的属性，一个属性可能是一个任务的描述或者是项目的版本。


#### 扩展属性

使用命名空间 ext 自定义属性

```
// 定义只读属性
project.ext.name = 'name'
// 定义可读写属性
ext{
    age = 1
}
ext.one = 2
```

#### Gradle 属性


在 gradle.properties 文件中可以添加 Gradle 属性，在 .gradle 文件下的 gradle.properties 中添加的 Gradle 属性，所有的项目都可以使用，在项目下的 gradle.properties 中添加的 Gradle 属性，可以在项目中所以的模块中使用。



