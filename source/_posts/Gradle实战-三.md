---
title: Gadle实战(三)：自定义 Task 类
date: 2020-01-16 19:02:23
tags: [Gradle 基本原理,Gradle in action]
---

在构建脚本中可以编写 Task 的动作（Action），从而操作相关逻辑，这在 Gradle 中是十分简单的。但是当项目增长需要添加更多的逻辑时，维护起来就十分的麻烦，如果可以编程式结构实现对逻辑的编写，类似对类和方法编写，按照日常开发的思维编写代码，这对于程序开发是十分友好的，而 Gradle 提供了这种支持，开发者可以使用任意 JVM 语言，比如 Java、Groovy、Kotlin，在构建脚本中进行编码。

## 1. 自定义 Task

自定义 Task 包含两个组件：

<!-- more -->
1. 自定义 Task

自定义的 Task 类，封装了逻辑行为，被称为 **任务类型**。

2. 真实的 Task 

提供用于配置行为的 Task 类所暴露的属性值。

Gradle 把以上 Task 称为 **增强的 Task**，具体的自定义 Task 可以参见官方文档：[Developing Custom Gradle Task Types](https://docs.gradle.org/current/userguide/custom_tasks.html#header)。

## 2. 编写自定义的 Task 类

* 继承 DefaultTask 的类。
* 通过注解声明 Task 类的输入和输出：@Input、@OutputFile

一个简单的示例：

```
class ReleaseVersionTask extends DefaultTask{
    // 通过注解声明 task 的输入和输出
    @Input Boolean release
    @OutputFile File destFile
    // 在构造器中设置 task 的 group 和 description 属性
    ReleaseVersionTask(){
        group = 'versioning'
        description = 'Make project a release version.'
    }
    // 通过注解声明被执行的方法，动作方法
    @TaskAction
    void start(){
        project.version.release = true
        ant.propertyfile(file: destFile) {
            entry(key: 'release', type: 'string', operation: '=', value: 'true')
        }
    }
}
```
## 3. 使用自定义 Task 类

定义一个 **增强** 的 `ReleaseVersionTask` 类型的 Task，通过 type 声明该 Task **派生于** `ReleaseVersionTask。`

```
task makeReleaseVerson(type:ReleaseVersionTask){
    release = 'true'
    destFile = file('version.properties')
}
```

以上实例可以认为在 **创建一个特定类（ReleaseVersionTask）的新实例**，并在构造器中为它的属性值设置初始值。

## 4. Gradle 的内置 Task

Gradle 中提供了大量的内置 Task， 它们均派生于 DefaultTask，所以可以在脚本中被增强 Task 使用。

比如，Copy 为 Gradle 内置 Task，以下展示将 createDistribution 的输出文件复制到指定目录中。

```
task createDistribution(type: Zip,depensOn: makeReleaseVersion){
    // 隐式引用了 War task 的输出
    from war.output.files   
    // 把所有的源文件放到 ZIP 文件的 src 目录中
    from(sourceSet*.allSource){
        into 'src'
    }
    //为 Zip 文件添加版本文件
    from(rootDir){
        include versionFile.name
    }
}

task copyFile(type:Copy){
    // 使用 Copy 的属性，隐式调用 createDistribution 的输出
    from createDistribution.outputs.files
    into "$buildDir/backup"
}
```

Zip 和 Copy 都继承与 AbstractCopyTask，具体可以查看相应的 API。

## 5. Task 规则

Gradle 引入了 Task 规则的概念，可以根据 Task 名称模式执行相应的逻辑，该模式是有两部分组成：Task 名称的静态部分和一个占位符，它们联合组成动态的 Task 名称。

根据 Task名称模式执行相关逻辑，并不意味着你可以不做任何工作只通过 Task 名称来执行相关的逻辑，你需要自定义 Tast 规则，至于如何定义 Task 规则，参见 《Gradle in action》一书中第 98 页。 


## 6. 在 buildSrc 目录下构建代码

**buildSrc 目录被视为 Gradle 项目的指定目录**，这个目录是十分重要。


在构建脚本中编写的 Groovy 类最适合的位置为项目的 buildSrc 目录下。将 Java 代码放在 src/main/java 目录下，将 Groovy 代码放在 src/main/groovy 目录下，**位于这些目录下的代码会被自动编译，并且会被加入到 Gradle 构建脚本的 classpath 中**，所以说 buildSrc 目录是组织代码的最佳方式。

在 buildSrc 中构建代码的用途是十分重要的，我们常见的一个需求：开发一个本地插件，我们不需要将开发的插件发布到第三方库中，只需要在该项目中使用新建 buildSrc 目录，那么我们就可以在该目录内进行编码，如上面所说该目录下得文件是可以被 Gradle 自动识别的。


## 7. 挂接到构建生命周期过程中

Task **的动作和配置代码是在构建生命的不同阶段执行的**。

在构建阶段执行的 Task 配置逻辑，或者在执行阶段执行的 Task 动作的操作，在很多时候是存在局限的，很多时候需要在特定的生命周期事件发生时执行指定的代码，比如在某个构建之间、期间或者之后。对于这种要求 Gradle 提供了两种方式可以编写回调生命周期的事件：
* 在闭包中调用生命周期钩子
* 通过 Gradle API 提供的监听器接口实现


许多生命周期的回调方法被定义在 Project 和 Gradle 的接口中。

Task 执行图是一个 **有向无环图 (DAG)**，**一个执行过的 Task 永远不会再次执行**,以下展示了 Task 执行图的相关接口和方法：

![Task 接口](Gadle实战-三/2020_01_17_02.png)

### 7.1 在闭包中调用生命周期钩子

以 whenReady 为例，whenReady 方法会在 task 图生成完成后，该函数会立即被执行。

```
// 注册的生命周期钩子函数在 Task 图生成后被调用
gradle.taskGraph.whenReady{ TaskExecutionGraph taskGraph ->
    // 查看Task 执行图中是否含有 release Task
    if(taskGraph.hasTask(release)){
            // 执行相关的逻辑
    }
}
```

### 7.2 通过 Gradle API 实现 Task 执行图监听器


通过监听器挂接到构建生命周期只需要两个步骤：

1. 在构建脚本中编写一个类来实现特定的监听器接口

用于监听 Task 执行图的事件的接口是 TaskExectionGraphListener 接口提供：

2. 注册监听器，实现监听

```
// 实现相应的接口
class ReleaseVersionListener implements TaskExecutionGraphListener {

    final static String releaseTaskGraph = ':release'

    @Override
    void graphPopulated(TaskExecutionGraph graph) {
        // release task 在执行图中
        if (graph.hasTask(releaseTaskGraph)) {
            List<Task> allTasks = graph.allTasks
            // 从一系列的执行图中找到 release Task
            Task releaseTask = allTasks.find{it.path == releaseTaskGraph}
            // 每个 Task 都知道自己所云的 project
            Project project = releaseTask.project
            if (!project.version.release) {
                println "i am in listener"
                // 显式的调用getProject
                project.version.release = true
                project.ant.propertyfile(file: versionFile) {
                    entry(key: 'release', type: 'string', operation: '=', value: true)
                }
            }
        }
    }
}
// 注册接口的实现
def releaseVersionListener = new ReleaseVersionListener()
// 将注册对象添加到检测列表中
gradle.taskGraph.addTaskExecutionGraphListener(releaseVersionListener)
```

那么在 Gradle 在v监听到 :release 任务后，会执行自定义逻辑。
---

**知识来源**：


[实战 Gradle ](https://e.jd.com/30505980.html)，一本翻译 “感人” 的书，都是泪。