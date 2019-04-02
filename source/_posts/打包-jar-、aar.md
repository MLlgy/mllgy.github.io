---
title: 打包 jar 、aar
date: 2019-03-27 10:47:06
tags:
---

### 概念

aar:(Android Archive) 是一个 Android 库项目的二进制归档文件，里面不仅包含工程代码还包括工程资源文件，大部分的 aar 文件包括：AndroidManifest.xml，classes.jar，res，R.txt。

jar：只包含了 class 文件和清单文件，不包含资源文件。

如果需要资源文件，那么以 aar 的形式引入到工程，反之使用 jar。


### 打包 aar、jar

如果将 Application 打包为 aar，做以下更改：

1. 将 `apply plugin: 'com.android.application'` 改为 `apply plugin: 'com.android.library'`
2. 去掉 `applicationId`
3. 项目根目录执行 `./gradlew assembleRelease `,就可以在相应的目录(build/output/aar)下看到生成的 aar，在 `build/intermediates/packed-classes` 中看到相应的 jar 包。


可以使用新建 gradle task 可以将生成的 jar 包直接复制到 libs 下，并完成构建。
```
task copyJar(type: Copy) {
    def name = project.name //Library名称
    delete 'libs/' + name + '.jar' //删除之前的旧jar包
    from('build/intermediates/packaged-classes/release/') //从这个目录下取出默认jar包
    into('libs/') //将jar包输出到指定目录下
    include('classes.jar')
    rename('classes.jar', name + '.jar') //自定义jar包的名字
}
copyJar.dependsOn(build)

```


### 自定义 jar
