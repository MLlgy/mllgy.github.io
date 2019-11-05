---
title: git submodule 使用
tags: [Git]
date: 2019-11-05 12:22:23
---


### 1. 为项目添加子模块


> git submodule add <repository> <path> 

此命令会生成 .gitmodule 文件以及在 .git/config 文件中添加子模块的相应信息

### 2. 克隆一个带有子模块的项目

<!-- more -->

**方式一：**
> git clone xxxx

这是会发现子模块的内容并未被克隆，只有对象模块的文件夹，通过以下命令可以查看子模块的状态：

> git submodule status

```
-01xxxxx lib/lib_a
-f25xxxx lib/lib_b
```

最前面的减号的含义是该子模组未被检出。

如果想要检出子模组的外部库，需要执行以下命令：

> git submodule init

此命令会生成 .gitmodule 文件以及在 .git/config 文件中添加子模块的相应信息。

然后执行以下命令完成子模组的克隆：

> git submodule update


**方式二：**

可以直接使用如下命令，在克隆项目的同时递归克隆子模块组：

> git clone [reps] [path] --recursive


### 修改子模组以及子模组的更新


**修改子模组**

在子模组版本库的工作区中修改相应内容，之后提交远端库，修改子模组库对主项目库不会有影响。


此情况下，推荐先推送子模组，在推送父版本库。

**更新子模组库**

在子模组库中拉取子模组远端库最新提交。

### 删除子模组

> git rm --cached xxx
> rm -rf xxx
> rm .gitmodules

移除 .git/config 文件中关于子模组的配置。

---

[Git 中 .git/config 中的属性含义](https://git-scm.com/docs/git-config)