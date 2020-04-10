---
title: Gradle 之 api compileOnly
tags:
---


### implemention

在模块编译时不会暴露该模块的依赖项，其他模块只有在运行时才可以访问该模块的依赖项。
### api

当模块通过 api 引入其他依赖项时，那么该模块会以传递的方式将自己的依赖项导出给其他模块，但是在运行期这些依赖项可以被其他模块访问。



### compileOnly  使用场景


* 运行时不需要，例如仅源代码注解或注释处理器;
* 编译时仅需要其API，但是其他 module 也会引入该依赖，那么可以使用 compileOnly ，避免在编译时报出 classNotFound 的错误，在运行期 app 中其中的一个 module 含有该依赖就可以了。


所以 compileOnly经常用于解决依赖冲突等问题，一般第三方库中，比较常用的依赖，如support、gson、Eventbus等等。


[compileOnly的使用场景](https://www.jianshu.com/p/825004db000c)


使用 compileOnly 引入的依赖项， Gradle 只会将依赖项添加到编译类路径，也就是说该依赖项只会在该 module 的编译期间可以使用，在运行期无法获得该依赖的相关的类，具体使用场景见上面。


### runtimeOnly

Gradle 只会将依赖项添加到构建路径上，也就是说该依赖项只会在运行期可用，在编译期无法使用该依赖项的相关类。


### annotationProcessor	

如果添加的依赖为注解处理器相关的库，那么必须使用 annotationProcessor 配置该远端库。
使用此配置可以将编译类路径与注释处理器类路径分开，从而提高构建性能。
这是因为，使用此配置可以将编译类路径与注释处理器类路径分开，从而提高构建性能。如果 Gradle 在编译类路径上找到注释处理器，则会禁用避免编译功能，这样会对构建时间产生负面影响（Gradle 5.0 及更高版本会忽略在编译类路径上找到的注释处理器）。


如果 JAR 文件包含以下文件，则 Android Gradle 插件会假定依赖项是注释处理器：
META-INF/services/javax.annotation.processing.Processor。 如果插件检测到编译类路径上包含注释处理器，则会生成构建错误。


---

学习连接：

[Gradle 与 Android 构建入门](https://mp.weixin.qq.com/s/HdCrhiY3VSsEjmu0FKLlyg)