---
title: Gadle实战(一)：使用 Task
tags:
---


## 依赖配置

插件可以引入配置来定义依赖的作用域。


比如 Java 插件引入了各种标准配置来定义 Java 构建生命周期所应用的依赖


## 配置 API

配置可以在项目的根级别添加和访问，可以使用插件提供的配置，也可以使用自己声明的配置。

每个 Project 都有一个 ConfigurationContainer 类容器来管理自己相应的配置。

配置是很灵活的，可以使用配置控制配置依赖解决方案追溯是否包含传递性依赖、定义解决策略（比如，如何解决版本冲突）等，同时使用配置可以进行


## 自定义配置



## 声明依赖

DSL 配置块 dependencies 通常用来将一个或者多个依赖指派给配置。

### 依赖的方式

外部模块依赖（dependencies）、项目依赖（settiong）、文件依赖（dependencies 中 fileTree）、Gradle 运行时依赖、客户端模块依赖（不常见）

### 依赖相关 API

每个 Gradle 项目都有依赖处理器实例，由 DependencyHandler 接口来实现。

每个依赖项都是 Dependency 类型的一个实例，group、name、version、classifier 属性明确标识了一个依赖。


### 外部模块依赖

在 Gradle 中，外部类库通常以 Jar 文件的形式存在，被称为 **外部模块依赖**。

`androidx.appcompat:appcompat:1.1.0`

依赖属性：

* group
这个属性用来标识一个组织、公司或者项目，比如 androidx.appcompat

* name

一个库的名称唯一描述了依赖，比如 appcompat

* version

描绘了一个库的版本号。

* classifier

可以定义另一个属性，但是以上并没有定义该属性。


### 依赖标记


在项目中通过以下形式来声明依赖：

dependencies{
    configurationName xxxxx,xxxx
}

### 检查依赖报告

运行 `gradle dependencies` 会显示完整的 **依赖树**。

在依赖树中，标有星号的依赖被排除了，这意味依赖管理器使用的是另外一个版本的类库，因为他被声明另外一个顶层依赖的传递性依赖。

在 dependencies 配置中声明的依赖为顶层依赖，这些顶层依赖所依赖的库称为传递性依赖。

**针对版本冲突，Gradle 默认的解决策略是获取最新的版本**，在依赖树中这样展示：1.0.0 -> 1.1.0.

### 排除传递性依赖

#### 排除传递性依赖，依赖指定版本的远端库

假如想要显式的指定某个类库的版本，而不是使用顶层依赖所提供的传递性依赖，如下面：

```
dependencies{
    // 通过 exclude 来声明排除依赖
    cargo('org.codehaus.cargo:cargo-ant:1.3.1'){
        exclude group:'xml-apis',module:'xml-apis'
    }
    // 依赖指定版本的库
    cargo 'xml-apis:xml-apis:2.0.2'
}
```

#### 排除传递性依赖，排除所有的传递性依赖

如果想要排除一个库的所有传递性依赖，Gradle 提供了 transitive 属性来实现这一效果。

```
dependencies{
    // 通过 exclude 来声明排除依赖
    cargo('org.codehaus.cargo:cargo-ant:1.3.1'){
        transitive = false
    }
}
```


### 动态版本声明

如果不想要指定依赖库的版本号，可以获取最新版本的依赖或者在版本范围内选择最新的依赖。

动态版本声明有特定的语法，如果想要使用最新版本的依赖，则必须使用占位符 `lastest.intergration` 或者声明版本属性，通过使用一个加号（+）标定它来动态改变。

```
dependencies{
    cargo 'xml-apis:xml-apis:lastest.intergration'
    cargo 'xml-apis:xml-apis:2.0.+'
}
```

**不要最好不要使用动态版本，因为在项目的开发中，可靠性和可复用是最重要的，但是选择最新版本的类库可能在开发者不知情的情况下引入了不兼容的类库版本和副作用，基于此应该声明明确的类库版本。**


### 文件依赖

以上为外部模块依赖，也可以依赖本地的文件。

这样一个场景：定义一个 Task，用于将从 Maven Central 获取的依赖拷贝到 home 目录下的 libs/cargo 子目录中。

```
task copyDependenciesToLocalDir(type:Copy){
    from confingurations.cargo.asFileTree
    into "${System.properties['user.home']}/libs/cargo"
}
```
在 dependencies 配置块中声明 Cargo 类库，以下展示如何把 Jar 文件指派给 cargo 配置作为文件依赖。

```
dependencies{
    cargo fileTree (dir:"${System.properties['user.home']}/libs/cargo",include '*.jar')
    // 平时我们在 Android 如下使用，进行文件依赖
    implementation fileTree(dir: 'libs', include: ['*.jar'])
}
```

## 使用和配置仓库

### 仓库 API

在项目中定义仓库的关键是 RespositoryHandler 接口，**该接口提供了添加各种类型仓库的方法**，这些方法可以在 respositories 配置块中被调用。

![](/source/images/2020_01_14_01.png)

通过上图可以看，不同类型的仓库提供了不同方法来进行相关配置，以下展示每种仓库的具体配置。

### Maven 仓库

最常的仓库，类库通常以 jar 文件的形式表现，元数据用 xml 标出，并且使用 pom 文件描述了类库相关信息以及其传递性依赖。

Maven 仓库下所有类库的内容会被存储在一个预定义的目录结构中，在声明一个依赖时，依赖的属性（group、name、version）来确认它在仓库中的位置。

![](/source/images/2020_01_14_02.png)

Maven 提供了两个方法用来配置 Maven 仓库：

* mavenCentral()

将 Maven Central 引用添加到一系列的仓库中。

```
repositories{
    mavenCentral()
}
```

* mavenLocal()

在文件系统中关联一个本地的 Maven 仓库，本地 Maven 仓库的默认目录为：<USER_HOME>/.m2/repository。
```
repositories{
    mavenLocal()
}
```


### 自定义自定义 Maven 仓库

很多情况下，我们需要配置企业级的仓库来完成相应的开发，仓库管理器提供类了一个以 Maven 结构来配置仓库的功能，Gradle API 提供了两个方式来配置自定义仓库：

* maven()

```
repositories{
    maven{
        name 'custom maven repository'
        url 'http://xxx/xxx/release/'
    }
}
```

* mavenRepo()


### Ivy 仓库

Maven 仓库中的文件必须以一个固定的布局存储，任何文件结构的偏差都有可能导致依赖关系发生变化，而 Ivy 仓库则可以完全自定义默认布局。

在 Ivy 里，仓库依赖被存储在 ivy.xml 文件中，Gradle 提供了各种方法来配置 Ivy 仓库及特定的文件布局。


声明 ivy 仓库

```
repositories{
    ivy{
        // Ivy 仓库的基础 URL
        url 'http://xxx/xxx/release/'
        layout 'pattern',{
            // 工件模式
            artifact '[xx]/[xxx]/[version].[ext]'
            // 元数据模式
            ivy '[xx]/[xxx]/[version]/ivy-[revision].xml'
        }
    }
}
```

### 扁平的目录仓库

flat 目录仓库是最简单和最基本的仓库形式，在文件系统中它是一个单独的目录，只包含了 jar 文件，没有元数据。

当声明此种仓库依赖时，只能使用 name 和 version 属性，不能使用 group 属性，因为它会产生不明确的依赖关系。

以下通过 map 和 快捷方式来声明从 flat 目录仓库中获取 Cargo 依赖。

```
// 声明扁平的目录仓库
repositories{
    flatDir (dir:"${System.properties['user.home']}/libs/cargo",name :'Local lins directory')
}
dependencies{
    // 以 map 的形式声明依赖
    cargo name:'activation',version:'1.1.0'
    // 以快捷方式获取依赖
    cargo ':jaxb-api:2.1.1' 
}
```

需要手动的声明传递性依赖，需单独的声明每个依赖，这种方式耗时耗力。


## 理解本地依赖缓存

以上阐述了如何声明各种类型的仓库，在执行的 Task 会自动确定所需要的依赖，在执行时从仓库中下载工件，并将它们存储在本地缓存中。在之后的任何构建都会重用这些工件。


此节将深入分析缓存结构，确定缓存在底层是如何工作的，以及如何调整其行为。

### 分析缓存结构


执行 Task ，Gradle 下载的 jar 文件被存放在何处？通过以下操作，打印出完整的、指派个 cargo 配置的所有依赖的连接路径。

```
task printDependencies{
    doLast{
        configurations.getByName('cargo').each {depency ->
            println depency
        }
    }
}
```
执行该 Task 会发现所有的 jar 文件都存储在 .gradle/caches/artifacts-15/filestore 目录中，而 artifacts-15 为一个标识符，用来指定Gradle 版本，同时需要注意的是这个文件目录结构在不同的Gradle 中可能不同。


缓存其实包含两部分：

* filestore 中的文件

该目录中包含了从仓库下载的原始二进制文件。

* 其他二进制文件

这些文件中存储了已下载工件的元数据。


[The Directories and Files Gradle Uses](https://docs.gradle.org/current/userguide/directory_layout.html#dir:gradle_user_home)



---
```
├── caches // (1) 全局缓存目录
│   ├── 4.8 // 特定版本的缓存，支持增量更新
│   ├── 4.9 // (2)
│   ├── ⋮
│   ├── jars-3 // 共享缓存，例如对于依赖项的工件
│   └── modules-2 // (3)
├── daemon // (4)Gradle 守护进程的注册表和日志
│   ├── ⋮
│   ├── 4.8
│   └── 4.9
├── init.d // 全局初始化脚本
│   └── my-setup.gradle
├── wrapper
│   └── dists // 由Gradle包装器下载的发行版
│       ├── ⋮
│       ├── gradle-4.8-bin
│       ├── gradle-4.9-all
│       └── gradle-4.9-bin
└── gradle.properties // 全局层次配置属性
```

支持在规定时间删除缓存和版本。同样见上面链接。


## 解决依赖问题


如果选择自动解决传递性依赖，那么版本冲突是不可避免的，Gradle 解决版本冲突的默认策略是选择最新的依赖版本。


依赖报告是十分有用的工具，它可以帮助我们选择需要的依赖版本。

### 修改默认的策略

做如下修改，当遇到版本冲突时，会让构建失败,此时在打印台上你可以很清楚的指导那些传递依赖库发生了版本冲突。

```
configurations.all{
  resolutionStrategy{
      // 修改gradle不处理版本冲突
      failOnVersionConflict()
  }
}
```
此处为所有的配置重新配置策略，也可以为指定的配置重新配置冲突解决策略，比如上文一直使用的 cargo。


```
configurations.cargo.resolutionStrategy{
    failOnVersionConflict()
}
```

### 强制指定一个版本

```
configuration.all{
  resolutionStrategy{
      force 'org.slf4j:slf4j-api:1.7.24'
  }
}
```
此处为所有的配置重新配置策略，也可以为指定的配置重新配置冲突解决策略。

```
configuration.cargo.resolutionStrategy{
    force 'org.slf4j:slf4j-api:1.7.24'
}
```

### 依赖观察报告

依赖观察报告，解释了依赖图中的依赖是如何选择的以及为什么。

为了运行这个报告需要指定两个参数：配置名称（默认是 compile）和依赖本身。

> gradle -q dependencyInsight --configuration cargo --dependency xml-apis:xml-apis

dependencyInsight:显示原因


使用 `gradle dependencies` 生成的依赖报告，是从顶层依赖开始的，而此处的依赖观察报告则与之相反，是从低层到顶层的。


[Gradle 命令行选项含义](https://docs.gradle.org/current/userguide/command_line_interface.html#common_tasks)


### 刷新缓存


针对依赖的 SNAPSHOT 版本和使用动态版本的模式声明依赖，Gradle 一旦获取了依赖，它们会缓存 24 小时，在缓存时间到后，会再次检查仓库，如果依赖库发生版本变化，Gradle 会下载最新的依赖库。


可以使用命令行选项 --refresh-dependencies 手动刷新缓存中的依赖，这个标志会强制检查配置仓库中所依赖的库版本是否发生了版本变化，如果变化则再次下载，取代缓存中的副本。


同时 Gradle 也支持配置缓存的默认行为，

* 设置缓存动态依赖版本 0 秒超时

```
configuration.cargo.resolutionStrategy{
    cacheDynamicVersionsFor 0,'second'
}
```

* 不缓存 SNAPSHOT 版本

```
configuration.cargo.resolutionStrategy{
    cacheChangingMoudlesFor 0,'second'
}
```
