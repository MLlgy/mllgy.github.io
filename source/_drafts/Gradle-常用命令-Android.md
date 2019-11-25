---
title: Gradle 常用命令(Android)
date: 2019-03-29 16:31:19
tags:
---



查看项目依赖树：
```
./gradlew :app:dependencies
```


### gradle 如何从远端仓库下载 jar/aar 包

1. 配置 gradle 仓库

```
repositories {
    mavenCentral()
    google()
    // 其他仓库
}
```
2. 在项目中添加三方库依赖


```
dependencies {
    implementation 'com.android.support:appcompat-v7:28.0.0'
    ...
}
```
3. 构建项目，下载依赖文件

默认从远端仓库下载的 Jar 包会保存在用户目录的 .gradle 目录下，比如 .gradle/caches/modules-2/files-2.1 、.gradle/caches/transforms-1/ 等相应目录下。

上文添加的依赖的下载的 jar 包的具体目录：

![](/source/images/2019_11_21_01.png)


在构建项目时，gradle 会首先根据依赖项到 .gradle 目录中查找相应的 jar/aar 包，如果有的话直接依赖，如果没有的话去远端仓库下载，进行依赖。



关于 Gradle 缓存文件目录可以查看 [Gradle 缓存目录结构 缓存策略](https://www.jianshu.com/p/acf579d8cb56)、[gradle cache目录(.gradle)剖析](https://zhuanlan.zhihu.com/p/26473930)

---

[Gradle之多版本打包不同依赖配置](https://my.oschina.net/jjyuangu/blog/1560107)

[官方文档：Gradle 提示与诀窍](https://developer.android.google.cn/studio/build/gradle-tips?hl=zh_cn)