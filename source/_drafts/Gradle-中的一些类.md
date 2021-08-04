---
title: Gradle  中的一些类
tags:
---


## Project


Project 接口是用于与构建文件中的 Gradle 交互的主要API。 通过 Project 接口，可以通过编程方式访问 Gradle 的所有功能。

### Lifecycle

Projcet 和 `build.gradle` 文件之间存在一对一的关系。 在构建初始化期间，Gradle 为要参与构建的每个项目组装一个Project对象，如下所示：

* 为一次 build 操作创建一个 Setting 实例
* 评估(Evaluate) `settings.gradle` 脚本，如果存在该脚本，根据该脚本配置 Setting 对象
* 使用配置后的 Settings 对象创建 Project 实例的层次结构
* 最后，执行每一个 Project 的 build.gradle 脚本(如果存在)，对 Project 进行评估(Evaluate).父 Project 在子 Project 之前被评估，但是调用 EvaluationDependsOnChildren（）或通过使用EvaluationDependsOn（String）添加显式评估依赖项来覆盖此顺序。


## Tasks

Tasks 本质上是一系列 Task 的集合，每个 Task 执行一些基本的工作，比如编译 class 文件、执行单元测试、压缩文件。在 Project 中通过 TaskContainer 的 create(xxx) 方法添加新的 Task，同时也可以通过 TaskContainer 的方法定位已经存在的 Task，比如 TaskContainer.getByName(string)。


## Dependencies

一个 Project 通常具有许多需要的依赖关系才能完成其工作。 同样，一个项目通常会产生许多其他项目可以使用的工件。 这些依赖项 **按配置分组** ，可以从存储库中检索和上载。 

* 可以使用 getConfigurations（）方法返回的 ConfigurationContainer 来管理配置。 
* getDependencies（）方法返回的 DependencyHandler 用于管理依赖项。
*  getArtifacts（）方法返回的ArtifactHandler用于管理工件。 
* getRepositories（）方法返回的RepositoryHandler用于管理存储库。


## Multi-project Builds


项目 被安排在项目的层次结构中。 项目有一个名称，以及在层次结构中唯一标识该项目的标准路径。


## Plugins

插件可用于模块化和重用项目配置。 可以使用PluginAware.apply（java.util.Map）方法或使用PluginDependenciesSpec插件脚本块来应用插件。

## Dynamic Project Properties


Gradle 会执行模块中的构建文件，以此对 Project 实例进行配置。 构建脚本中使用的任何属性或方法都将委托给关联的 Project对象，这意味着可以直接在脚本中使用 Project 接口上的任何方法和属性。

🌰：

```
defaultTasks('some-task')  // Delegates to Project.defaultTasks()
reportsDir = file('reports') // Delegates to Project.file() and the Java Plugin

```

可以在脚本中直接通过 project 关键字获取 Project 实例对象，


在一个项目中有 5 个用来搜索属性的 “作用域”，在构建文件中，可以通过 name 访问这些属性，也可以调用 Project 的 property(string) 方法。以下为属性的作用域：

* Project对象本身。此范围包括由 Project 的实现类声明的所有属性的 getter 和 setter 方法。例如，getRootProject（）可作为 rootProject 属性访问。该范围的属性是可读写的，具体取决于相应的吸气剂或设置器方法的存在。

* 项目的额外属性。每个项目都维护一个额外属性的 Map 集合，其中可以包含任何任意名称->值对。定义后，此范围的属性是可读和可写的。有关更多详细信息，请参见其他属性。

* 插件添加到项目中的扩展（Extension）。每个扩展名都可以作为只读属性使用，其名称与扩展中的属性名相同。

* 插件将约定属性添加到项目中。插件可以通过项目的 Conventional 对象向项目添加属性和方法。此范围的属性可能是可读或可写的，具体取决于约定的对象。

* 项目的 Task。Task 的名称可以作为属性用来访问该 Task。该范围的属性是 **只读**的。例如，可以将名为 compile 的Task 作为 compile 属性进行访问。

* 额外的属性和约定的属性从项目的父级继承，直到根项目为止。该范围的属性是只读的。


读取属性时，Project 将 **按顺序** 搜索上述范围，并从找到该属性的第一个范围返回值。如果找不到，则会引发异常。 有关更多详细信息，请参见 property（String）。

编写属性时，项目将按顺序搜索上述范围，并在找到该属性的第一个范围内设置该属性。如果未找到，则会引发异常。 有关更多详细信息，请参见setProperty（String，Object）。

## Extra Properties

Extra Properties 必须通过 ext 命名空间定义，一旦定义了额外属性，它就可以直接在拥有的对象上使用（在下面的情况下分别是Project，Task和sub-projects），并且可以读取和更新。 仅需要通过名称空间完成初始声明。


```
project.ext.prop1 = "foo"
 task doStuff {
     ext.prop2 = "bar"
 }
 subprojects { ext.${prop3} = false }
```


## Dynamic Methods


一个项目有5个方法“作用域”，它会搜索方法：

* Project对象本身。
* 构建文件。该项目搜索在构建文件中声明的匹配方法。
* 插件将扩展添加到项目中。每个扩展都可以作为一种方法，该方法以闭包或 Action 作为参数。
* 插件将约定方法添加到项目中。插件可以通过项目的Conventional对象向项目添加属性和方法。
* 项目的任务。为每个任务添加一个方法，使用任务名称作为方法名称并采用单个闭包或Action参数。该方法使用提供的闭包调用关联任务的Task.configure（groovy.lang.Closure）方法。例如，如果项目有一个名为compile的任务，则添加一个带有以下签名的方法：void compile（Closure configureClosure）。
* 父项目的方法，递归直到根项目。
* 项目的属性，其值为闭合值。闭包被视为方法，并使用提供的参数进行调用。该属性的位置如上所述。




## Configuration


一个 Configuration 对象代表一组 artifacts 以及它们的依赖项。




























## 0x0000 Provider

* Provider 为一个提供特定类型值的容器对象，可以通过 get() 或者 getOrNull()  获得该对象持有的值。Provider 对象并不是一直持有可用的值，存在为未来的某个节点该值才会可用的情况，如果持有的值不可用，那么 isPresent() 的返回值为 false，此时获取值的操作会产生异常。

* Provider 对象持有的值并不是一成不变的。

* Provider 对象可以作为 Task（TaskA） 的输出，此时该 Provider 持有该 Task 的输出消息，如果该 Provider 同时作为另外一个 Task(TaskB) 的输入，此时 Gradle 会自动添加 TaskA 和 TaskB 的依赖关系，TaskB 依赖 TaskA。

* Provider 的一个典型用途是在 Gradle 的元素之间传值，比如 project extension 和 task 之间、Task 之间。
* Provider 允许将繁重的计算推迟到实际需要它们的值之前，这个时机通常为 Task 执行时。
* Provider 的持有的值必须是可变的


# 0x0001 创建 Provider 的途径

* Gradle 中的一些类型，比如 Property，该类继承了 Provider，可以被当做 Provider 直接使用。
* 调用 map(Trandformer)，从现有的 Provider 创建新的 Provider。
* `org.gradle.api.tasks.TaskContainer#register(String)` 的返回值，这个 Provider 实例代表这个 Task 实例对象。



# 0x0003 Property

和 Provider 的第 1 、3 条相同。


可以通过 `org.gradle.api.model.ObjectFactory#property(Class)` 创建 Property 实例对象，Property 的不同子类可以通过 `ObjectFactory` 的不同方法获得，比如 listProperty(class)