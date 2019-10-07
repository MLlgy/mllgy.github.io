---
title: Git 命令学习
tags:
---


### git diff


1. 不带任何选项和参数调用 git diff 显示工作区的最新改动，即 **工作区与提交暂存区相比的差异**。


2. 通过 HEAD 参数，显示 **工作区和 HEAD(当前工作分支)的差异**。

>$ git diff HEAD

3. 通过 --cached 或 --staged 调用 git diff 命令，可以看到 **提交暂存区和版本库文件中的差异**。

>$ git diff --cached


### git status -s

使用 git status -s 以简洁模式显示状态：

>$ git status -s
M test.txt

如果此时修改 test.txt ，执行 `git status -s` 显示简洁信息，那么此时的输出为：
MM test.txt


两个 M 的位置是不同的(两列)，其含义也是不同的：

* 第一个 M 的含义是：**版本库** 中文件与 **暂存区** 中的文件相比有改动。
* 第二个 M 的含义是：**工作区** 中的文件与 **暂存区** 中的文件相比有改动。


### git clean -fd

清除当前工作区没有假如版本库的文件和目录(非跟踪文件和目录)



### git cat-file 

提供 commit 对象的基本信息(ID、大小等)

-t: commit id 的类型
-p: commit id 对提交信息

其他参数，使用 --help 查看并使用。

### git rev-parse  

显示引用对应的提交 ID。

>$ git rev-parse master