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


### 

1. 查看提交日志记录
> git log --oneline -6

![](/../images/2019_07_27_01.png)


2. 假设在提交 D 中做出了错误的更改，我们现在下次操作抹去提交　Ｄ　以及操作　Ｄ所作出的文件更改。

那么我们有两种方法：

 1. 直接抹去提交　Ｄ，即操作提交　Ｃ　指向操作　Ｅ。

为了抑郁标记每次提交，我们对每次提交进行　tag 标记。

```
git tag F
git tag D HEAD^^
git tag C HEAD^^^
git tag B HEAD~4 
git tag A HEAD~5
```

通过日志可以查看被标记的　６　个提交。

> git log --oneline --decorate=6

![](/../images/2019_07_27_02.png)


通过　git checkout 命令暂时将 HEAD 指向 C
> git checkout C

执行拣选操作将提价 E 在当前 HEAD 上重放，由于 master^ 和 E 为相同指向，所以

> git cherry-pick master^

上述操作可能出现 cherry-pick conflic，自己解决后执行：

> git add xxx
> git cherry-pick --continue

执行拣选操作将提价 F 在当前 HEAD 上重放，由于 F 和 master 有相同的指向，所以有：

> git cherry-pick master

如果有冲突执行上步相同操作，此时通过提交日志可以看到提交 D 以及不在了

![](/../images/2019_07_27_03.png)


以下操作为十分重要的操作，需要将 master 分之重置到新提交