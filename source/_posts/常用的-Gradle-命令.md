---
title: 常用的 Gradle 命令
date: 2019-03-28 16:37:33
tags:
---



### gradlew -q app:dependencies

出现依赖库版本冲突，一般会包如下错：
```
All com.android.support libraries must use the exact same version specification (mixing versions can lead to runtime crashes). Found versions 28.0.0, 25.2.0. Examples include com.android.support:animated-vector-drawable:28.0.0 and com.android.support:support-media-compat:25.2.
```

一般出现这种错的原因是自己依赖的库与其他依赖的库所依赖的库为同一个 group 的库，但是版本不同。

使用 **./gradlew -q app:dependencies** 可以看到自己项目依赖库的层级关系：

解决：使用 exclude 排除相应的库
```
api 'com.android.support:appcompat-v7:28.0.0'
api ("com.alibaba:arouter-api:1.4.0"){
    exclude group: 'com.android.support',module:'support-v4'
}
```