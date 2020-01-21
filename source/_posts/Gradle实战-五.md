---
title: Gadle实战(五)：扩展 Gradle
date: 2020-01-19 16:09:48
tags:
---



## 1.从零构建插件

Gradle 将插件分为两类：

* 脚本插件
* 对象插件

一个脚本插件是一个普通的 Gradle 构建脚本，可以被导入到其他构建脚本中。对象插件可以实现你学到的事情，需要实现 `org.gradle.api.Plugin` 接口。

<!-- more -->

对象插件的源代码通常放在 buildSrc 目录下，要么和项目在一起，要么是一个独立的项目，并且以 Jar 包的形式发布。

## 2. 定制 task 的实现选项

Gradle 中有多种定制 Task 的方式，最简单的一种是把它和构建代码一起放在构建脚本里,这种方式存在之前的文章。当触发一个 Task 时，**定义的 Task 会自动编译并添加到 classpath 中**。

另外一种方式就是将定制的 Task 放到项目根目录下的 `buildSrc` 目录中，遵从语言插件定义的源代码目录约定。**位于 buildSrc 目录下的定制的 task 类，会被所有的项目构建共享，并且在 classpath 中自动可用。**为了使定制的 Task 在多个项目中共享，可以将它们打成 jar 包，然后在构建脚本中的 classpath 中定义。

以下为定义 Task 的不同实现方式：

![](/source/images/2020_01_15_01.png)

### 2.1 在脚本中定制 Task

Gradle 提供了一个可以通过继承的默认实现：`org.gradle.api.DefaultTask`。


```
class customTask implements DefaultTask{
    xxxx
}
```

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

## 3. 使用和构建对象插件

打包定制的 Task 实现为 Jar 的方式的优缺点,具体参见 Gradle in Action 一书。


**对象插件** 可以灵活去封装高度复杂的逻辑，并且提供强大的扩展机制可以在构建脚本中定制它的行为。和定制 Task 一样，可以完全访问 Gradle 的公共 API 和工程模型。

Gradle 提供了开箱即用的插件，称为 **标准插件**，也可以通过 **第三方插件** 进行扩展。许多插件都是自包含的，这意味着它们要么依赖 Gradle 的核心 API，要么通过包装代码提供功能。更复杂的插件依赖于其他类库、工具或者插件提供的特性。

下图展示了插件在 Gradle 架构的位置:


![Gradle 插件架构](/source/images/2020_01_15_02.png)


在平常的开发中，我们经常使用 Java 插件来扩展项目功能。如下图 Java 插件特性所示，插件可以提供一个 Task 集合，并且整合到执行生命周期中，引入新的项目布局并提供有意义的默认值，添加属性来定制化它的行为，给依赖管理暴露对应的配置（比如平时使用到的 compile）。


![Java 插件特性](/source/images/2020_01_15_03.png)



**通过一行代码引入 Java 插件，就可以使用 Java 插件相应的功能来编译源代码、运行单元测试、生成报告，并将项目打包为Jar**。


标准插件也提供了很多通用功能，大部分能够满足开发者的需求。由社区或这开源组织开发的三方插件，可以用来给构建脚本增强非标准功能。

### 3.1 使用对象插件

在项目中同通过 `apply` 方法来配置项目，从而使用标准插件，该方式是 Project 对象提供的方式，存在一个类型为 Map 的参数 options。

标准插件的便利在于他们是 Gradle 运行时的一部分，用户不需要知道插件所依赖的类库，这些类库的位置为 Gradle 安装目录的 libs/plugins 下。

#### 3.1.1 通过名字使用插件

插件标识符一个简短的名字，通过插件的元信息提供，在项目中使用 Java 插件，直接传入键值对 plugin：’java‘：

```
apply plugins: 'java'
```

#### 3.1.2 通过类型使用插件
如果插件没有暴露名字，或者两个插件的命名冲突，那么可以类型来使用插件。

```
apply plugin:org.gradle.api.plugins.JavaPlugin
```

#### 3.1.3 使用外部插件

构建脚本并不知道外部插件的存在，需要将它放到 `classpath` 下。可以通过 buildScript 方法来做这件事，它定义了外部插件的位置、仓库和插件依赖。


在配置阶段，Gradle 中构建项目模型，连接插件构建逻辑。一旦插件下载完成，就会放置在本地缓存中，以便后续的运行可以使用到它们。

以下展示如何使用 MavenCentral 中的 tomcat 插件

```
buildScript{
    repostories{
        // 该插件的原始仓库
        mavenCentral()
    }
    dependencies{
        // 定义插件依赖
        classpath 'org.gradle.api.plugins:gradle-tomcat-plugin:0.9.7'
    }
}
```

## 4. 解析对象插件


以下图片显示了实现一个插件的几种选择：


![](/source/images/2020_01_16_01.png)


对于实现一个对象插件，有四个元素是十分重要的：

* 放置插件实现的位置


Gradle 在这方式十分灵活，代码可以发在构建脚本中，可以放在 `buildSrc` 目录下，也可以作为一个 **独立的工程** 被开发并且以 Jar 包的形式发布。

* 每一个插件都需要提供一个实现类

该实现类代表插件的入口，插件可以用任何 JVM 语言编写并编译成字节码，比如 Java、Kotlin、Groovy 等。

* 插件通过暴露出来的扩展对象进行定制

当用户想要在构建脚本中覆盖插件的默认配置，这是十分重要的实现方式。

* 插件描述符

插件描述符是一个属性文件，它包含类插件的元信息，通常包含插件的简短名字和插件实现类的映射。


## 5. 编写对象插件并运用到项目


编写一个插件的最低要求是提供 org.gradle.api.Plugin<Project> 接口的一个实现类，该接口仅有一个方法：apply(Project)。


### 5.1 通过 buildSrc 的形式编写对象插件

使用 buildSrc 的方式定义对象插件的好处是，在早期开发插件阶段，不需要打包插件代码，可以得到一个快速的反馈，能够让开发者能够专心通过 Gradle Api 实现业务逻辑。

在 buildSrc 工程下的指定包目录中创建一个插件的实现类，

```
class CustomPlugin implements Plugin<Project>{
    @Override
    void apply(Project project){
        xxx
    }
}
```
想要在项目中使用该插件，则在 build.gradle  中使用插件的实现类型：

```
apply plugin: xxxx.xxx.CustomPlugin
```


## 6. 插件扩展机制

通过 —P 和 —D 可以在执行 Gradle 命令行时提供参数，为 Task 提供输入，但这种方式不总是可取的。

Gradle 允许通过暴露一个带有唯一命名空间的 DSL 来建立自己的构建语言，下面展示一个名为 cloudBees 的闭包，允许从构建脚本中给 task 所需要的属性设置值。


```

cloudBees{
    apiUrl = 'https://xxx'
    apiKey = project.apiKey
}
```

Gradle 会将语言结构模型化为扩展，扩展是可以被添加到 Gradle 对象中，比如 Project或者Task。

如果一个类实现了 org.gradle.api.plugins.ExtensionAware 接口，就认为它是可扩展的，每种扩展都是一种数据结构，它是扩展的基础。

以下为 CloudBees 插件的扩展模型：

```
package xxxx.xxx
class CloudBeesPluginExtension{
    String apiUrl
    String apiKey
}
```
## 7. 注册和使用扩展

### 7.1 给插件一个有意义的名字

默认情况下，插件的名字从实现了 org.gradle.api.Plugin 接口的全限定类名继承而来。

对于对象插件来说，可以在 META-INF/gradle-plugins 目录下的一个属性文件中配置名字，**该属性文件的名字自动决定了插件的名字**，比如 META-INF/gradle-plugins/nuwa.properties 暴露插件的名字是 nuwa，在 nuwa.properties 中
需要将类的全限定类名赋值给 implementation-class,如下：

```
implementation-class=com.xxx.xxx.NuwaPlugin
```

在构建脚本 build.gradle 中使用这个插件：


```
apply plugin 'nuwa'
```


### 7.2 测试对象插件

具体查看 Gradle in action。

## 8.开发和使用独立的对象插件

如果想要在主构建的脚本中使用插件，那么在 buildSrc 项目中实现一个插件时是十分方便的；但是如果想要在多个模块中的共享插件，那么需要将插件作为独立的项目开发，然后将插件发布到仓库中。


### 8.1 项目和仓库配置


新建一个独立 module，将在 buildSrc 中的所有代码移到该  module 中。每次想要发布新的插件版本时，所产生的 Jar 文件会被发布到名为 repo 的与项目同目录的本地  maven 库中。

书中例子：假设其他项目想要使用该插件，那么在该项目的构建脚本中定义本地仓库，声明插件作为依赖，使用插件中的 Task 与 CloudBees 后端服务交互。


![](/source/images/2020_01_16_02.png)


其中 plugin 为定义插件的模块。

### 8.2 构建插件项目

* 定义依赖

此时不能再访问 buildSrc 基础设施，所以需要定义对 Groovy 和 Gradle API 类库的依赖。

* 发布插件

通过 Maven 插件，可以为插件生成的 POM 文件和将插件发布到 Maven 库中。配置 Maven 部署器将 POM 和 插件上传到本地目录中，为了更好的管理插件的版本，需要为插件指定 group、name\version.

![](/source/images/2020_01_16_03.png)


在插件被使用之前，需要先执行 `gradle uploadArchives` 命令，使用 Maven 插件相关 task 帮助下上传该插件。

发布插件后，在根目录下出现名为 repo 的新目录，包含了插件的 POM 文件和 Jar 文件。

* 在项目中使用插件


在项目中构建脚本中添加如下配置：

![](/source/images/2020_01_16_04.png)

对本地 Maven 库的声明也可以使用相对路径：

```
buildscript {
    repositories {
        jcenter()
        maven {
            url uri('./repo')
        }
    }
   ....
}
```