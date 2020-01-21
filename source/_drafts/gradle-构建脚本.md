---
title: gradle 构建脚本
date: 2019-04-17 17:39:10
tags:
---


[切换gradle 国内源](https://blog.csdn.net/superbeyone/article/details/86063787)

[自定义 Gradle 插件](https://deemons.cn/2017/10/16/%E8%87%AA%E5%AE%9A%E4%B9%89%20Gradle%20%E6%8F%92%E4%BB%B6/)

### 构建块

每个 Gradle 构建都包含 3 个基本的构建块：
1. project (项目)
2. task (任务)
3. property (属性)

[Gradle 之 DSL](https://blog.csdn.net/freekiteyu/article/details/81066845)：十分不错



###  常用

**classpath**：一般是添加 buildscript 本身需要运行的东西，buildScript是用来加载gradle脚本自身需要使用的资源，可以声明的资源包括依赖项、第三方插件、maven仓库地址等。
某种意义上来说，classpath 声明的依赖，不会编译到最终的 apk 里面。
