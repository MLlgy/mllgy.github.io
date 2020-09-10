---
title: Gradle 依赖冲突拾遗
date: 2020-07-02 14:06:43
tags:
---

## 0x0000 理解依赖解析

Dependency resolution is a process that consists of two phases, which are repeated until the dependency graph is complete(重复执行以下两个步骤，直到依赖图构建完成。):

* When a new dependency is added to the graph, perform conflict resolution to determine which version should be added to the graph(当新的依赖被添加到依赖图中，执行冲突解决方案，确定添加指定版本的库到依赖图中).

* When a specific dependency, that is a module with a version, is identified as part of the graph, retrieve its metadata so that its dependencies can be added in turn(当指定版本的库被标识为依赖图的一部分时，检索其元数据，依次添加其传递性依赖).

## 0x0001 Gradle 处理冲突

在以上解析依赖关系的过程中，Gradle 会处理两种类型的冲突：

* 版本冲突

两个或多个依赖项执行相同的依赖项，但是版本不同，在开发过程中比较常见。

* 实现冲突

That is when the dependency graph contains module that provide the same implementation, or capability in Gradle terminology(当 Gradle 依赖图中已经存在相同的功能或实现时。).


## 0x0002 解决版本冲突

版本冲突的解决方案：

* 选择指定版本
* 如果存在版本冲突，则会构建失败，由开发者自己指定库的版本

Gradle 的处理方法：Gradle 不关心该依赖出自依赖图的何处，它会考虑所有的版本，在这些版本中选择最高版本。

如何选择最高版本？具体参见知识链接


## 解决实现冲突

这个没接触过，接触后再做了解。


--


[Understanding dependency resolution](https://docs.gradle.org/current/userguide/dependency_resolution.html)
[Declaring Rich Versions](https://docs.gradle.org/current/userguide/rich_versions.html#header)