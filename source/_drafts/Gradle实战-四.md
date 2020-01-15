---
title: Gadle实战(一)：扩展 Gradle
tags:
---


##  从零构建插件


Gradle 将插件分为两类：


* 脚本插件
* 对象插件

一个脚本插件是一个普通的 Gradle 构建脚本，可以被导入到其他构建脚本中。对象插件可以实现你学到的事情，需要实现 org.gradle.api.Plugin 接口。对象插件的源代码通常放在 buildSrc 目录下，要么和项目在一起，要么是一个独立的项目，并且以 Jar 包的形式发布。




## 定制 task 的实现选项


### 在脚本中定制 Task

Gradle 提供了一个可以通过继承的默认实现：org.gradle.api.DefaultTask。

Gradle 中有多种定制 Task 的方式，最简单的一种是把它和构建代码一起放在构建脚本里，当触发一个 Task 时，**定义的 Task 会自动编译并添加到 classpath 中**。另外一种方式就是将定制的 Task 放到项目根目录下的 buildSrc 目录中，遵从语言插件定义的源代码目录约定。**位于 buildSrc 目录下的定定制的 task 类被所有的项目构建共享，并且在 classpath 中自动可用。**为了是定制的 Task 在多个项目中共享，可以将它们打成 jar 包，然后在构建脚本中的 classpath 中定义。


以下为定义 Task 的不同实现方式：



![](/source/images/2020_01_15_01.png)



### 在 buildSrc 中定制 Task

以上方式2，在 buildSrc 中创建定制 Task 的源文件，这是后期通过对象插件使用它们的最好方式。

用 groovy 定义 Task，所有的定制 Task 类放在 `${project_rootDir}/buildSrc/src/main/groovy/项目目录/` 下。

使用定制 Task：


```
//导入定制 Task
import xxx.xxxx.xxxx.CustomeTask
task test(type:CustomeTask){
    xxx
    xxx
}
```

## 使用和构建对象插件

打包定制的 Task 实现为 Jar 的方式的优缺点。


**对象插件** 可以灵活去封装高度复杂的逻辑，并且提供强大的扩展机制可以在构建脚本中定制它的行为。和定制 Task 一样，可以完全访问 Gradle 的公共 API 和工程模型。

Gradle 提供了开箱即用的插件，称为 **标准插件**，也可以通过第三方插件扩展。许多插件都是自包含的，这意味着它们要么依赖 Gradle 的核心 API，要么通过包装代码提供功能。更复杂的插件依赖于其他类库、工具或者插件提供的特性。

下图展示了插件在 Gradle 架构的位置:


![Gradle 插件架构](/source/images/2020_01_15_02.png)


在平常的开发中，我们经常使用 Java 插件来扩展项目功能。如下图 Java 插件特性所示，插件可以提供一个 Task 集合，并且整合到执行生命周期中，引入新的项目布局并提供有意义的默认值，添加属性来定制化它的行为，给依赖管理暴露对应的配置（比如平时使用到的 compile）。


![Java 插件特性](/source/images/2020_01_15_03.png)



通过一行代码引入 Java 插件，就可以使用 Java 插件相应的功能来编译源代码、运行单元测试、生成报告，并将项目打包为Jar。


标准插件也提供了很多通用功能，大部分能够满足开发者的需求。由社区或这开源组织开发的三方插件，可以用来给构建脚本增强非标准功能。