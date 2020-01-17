---
title: Gadle实战(零)：Gradle 的基本使用
date: 2020-01-16 18:04:11
tags: [Gradle 基本原理,Gradle in action]
---


## 1. Java 项目引入 Gradle 插件

当我们不通过 IDE 的方式构建 Java 项目，那么在编写完成后，运行项目就需要使用 javac 、jar 等这样的工具来帮我们运行项目，无疑在开发过程中这是十分痛苦的。

而 Gradle 插件具体自动化执行编译、运行等过程。

Gradle 通过引入特定领域的约定和 Task 来扩展项目。而 Java 插件是 Gradle 自身装载的一个插件，可以实现很多的功能，比如定义标准的项目布局、有顺序的执行任务。
<!-- more -->

如果想要使用 Java 插件，则在 build.gradle 中添加如下代码：

```
apply plugin:'java'
```

项目中配置类该插件，就可以构建编写的 Java 代码。Java 插件引入很多之一，其中之一是项目的源代码位置，默认情况下 Java 插件回到 src/main/java 目录下查找 Java 类。



## 2. 构建 Java 项目

在上文的基础上，构建创建的项目。Java 插件中的存在一个 build 的任务，以正确的顺序编译项目源代码，运行测试、组装 Jar 文件。

执行 `gradle build` 会得到相应的输出，每一行输出为 Java 插件提供的一个可执行任务，有些任务被标记了 UP-TO-DATE，这意味着这个任务被跳过了，因为  Gradle 支持增量式构建，不会运行不需要运行的任务。


## 3. 构建产生的文件


执行以上命令后，可以发现在根目录生成了一个 build 目录，该目录包含了构建运行时的所有输出，包括 jar 文件、测试报告和 class 文件，还有一些像清单一样对构建有用的临时文件。
构建输出的目录的名字是可以配置的，但是不建议自定义。


对 Java 项目完成构建后，可以运行该项目：

> java -cp build/classes.main com.xxx.xx.Main


-cp:告诉 Java 运行时去哪里找到 class。



## 4. 解析依赖


在构建脚本中添加依赖，如果依赖库没有被解挂过（成功下载过），那么就会在使用时去下载它，以下图片展示了依赖继续传递性依赖的过程：



![](/source/images/2020_01_16_05.png)



## 5. Gradle  包装器

Gradle 包装器是 Gradle 的核心特性，能够让机器在没有安装 Gradle 的情况下运行 Gradle 构建。在项目中配置包装器，需要两步操作：

1. 创建包装器任务

```
task wrapper(type:Wrapper){
    // 指定下载的 Gradle 版本
    gradleVersion = '5.4.1'
}
```

2. 执行任务

```
gradle wrapper
```

下载的文件如下：

![](/source/images/2020_01_16_06.png)

其中的 gradle-wrapper.properties 文件内容大致如下：

```
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-5.4.1-all.zip
```

此时，就可以使用包装器的脚本执行构建类。


## 6. 定制包装器


包装器是可以被重新配置的，指向运行有发布文件的企业内部服务器，同时还可以指定本地存储路径”


```
task wrapper(type:Wrapper){
    gradleVersion = '5.4.1'
    distributionUrl = ’http://companycenter.com/gradle/dists‘
    distributionPath = 'gradle-dists'
}
```