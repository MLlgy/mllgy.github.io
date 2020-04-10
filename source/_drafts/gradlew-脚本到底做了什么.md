---
title: gradlew 脚本到底做了什么?
tags:
---



首先看一下执行 


在 Android 项目目录下执行 `./gradlew clean` 最终的结果是执行了如下操作：

```
/Library/Java/JavaVirtualMachines/jdk1.8.0_191.jdk/Contents/Home/bin/java -Xdock:name=Gradle -Xdock:icon=/Users/liguoying/Git/IPCDemo/media/gradle.icns -Dorg.gradle.appname=gradlew -classpath /项目路径/gradle/wrapper/gradle-wrapper.jar org.gradle.wrapper.GradleWrapperMain clean
```

org.gradle.wrapper.GradleWrapperMain：指定的执行的类 


最终通过 java 命令，设置参数，执行 gradle-wrapper.jar，并传入参数 org.gradle.wrapper.GradleWrapperMain 和 clean


通过 -classpath 指定运行类所在的 jar 包，并且执行运行的类（org.gradle.wrapper.GradleWrapperMain）的 main 方法，并传入参数（clean）。



---
https://www.cnblogs.com/fhefh/archive/2011/04/15/2017613.html