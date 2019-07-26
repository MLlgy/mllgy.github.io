---
title: Git之改变历史提交
tags:
---


### 更改最新提交的 commit 信息(单步悔棋)

git-commit: Record changes to the repository

1. 查看最新提交的信息


> git log --stat -1

![](/../images/2019_07_26_01.png)

2. 使用 Git 修补式提交命令

> git commit --amend -m"change four"

![](/../images/2019_07_26_02.png)

也可以更改文件后，将执行上述命令修改最新的 commit 消息(不用执行添加暂存区操作)。


### 多步悔棋


可以取消最新连续的多次提交。

用途：可以将多个提交合成一个提交。

把最近两次提交压缩成一个

1. 查看版本库的最新三次提交

> git log --stat --pretty=onelin -3

2. 将 commit 恢复到两次提交之前

> git reset --soft HEAD^^

这时将 commit 恢复到两次提交之前，但是工作区不变，暂存区恢复到两次提交之前，引用也会回退两次之前。

3. 提交操作，此时完成将最新的两个提交压缩为一个新的提交。

> git commit -m"make two last commit into new one"

4. 查看日志，完成了多不悔棋

> git log --stat --pretty=onelin -3

![](/../images/2019_07_26_03.png)