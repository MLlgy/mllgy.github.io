---
title: Android 源码相关
tags:
---

## 1. AOSP 源码下载

[AOSP 源码下载](https://mp.weixin.qq.com/s?__biz=MzI4MTQyNDg3Mg==&mid=2247485238&idx=1&sn=f23f535732b90f02190276fcb607a29d&scene=21#wechat_redirect)


## Android 源码整编、单编

[AOSP 源码整编单编](https://mp.weixin.qq.com/s?__biz=MzI4MTQyNDg3Mg==&mid=2247485256&idx=1&sn=efc9e78a20b9d14f84c94a855c2cfef7&chksm=eba821cfdcdfa8d906cc6f7c948751ff24f85e8f172322be6375202419a65fcb44dbeb7f34c7&scene=21#wechat_redirect)


## 3. 查看源码


[Android Studio 导入 AOSP 源码](https://mp.weixin.qq.com/s?__biz=MzI4MTQyNDg3Mg==&mid=2247485337&idx=1&sn=05d6c8d8d4542688ec5a3d91c7da5da7&chksm=eba8211edcdfa808cab736a250fc5aaa5359690c1ee4f79c192e363b62b7ff17229c522f66d5&scene=21#wechat_redirect)


```
project.afterEvaluate(new Action<Project>() {
    @Override
    void execute(Project project) {
        com.android.build.gradle.AppExtension android = project.findProperty('android')
        android.getApplicationVariants().all(new Action<com.android.build.gradle.api.ApplicationVariant>() {
            @Override
            void execute(com.android.build.gradle.api.ApplicationVariant applicationVariant) {
                String buildTypeName = applicationVariant.buildType.name
                Task task = project.tasks.create("jar${buildTypeName.capitalize()}", Jar)//创建任务
                Task packageTask = project.tasks.findByName("package${buildTypeName.capitalize()}")//确定需要依赖的任务
                task.archiveName = 'base.jar'//输出文件名
                task.dependsOn(packageTask)//设置任务依赖的现有构建任务
                packageTask.finalizedBy(task)//设置任务在构建中的执行时机
                task.outputs.upToDateWhen { false }
                Collection<File> apkLibraries = applicationVariant.getApkLibraries()
                for (File file : apkLibraries) {
                    project.getLogger().lifecycle('apkLibraries ===> ' + file.getAbsolutePath())
                    task.from zipTree(file)
                }
                task.destinationDir = file(project.buildDir.absolutePath + "/outputs/jar")//输出目录
                artifacts.add('archives', task)
            }
        })
    }
})
```