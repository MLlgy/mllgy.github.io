---
title: 使用 git checkout 切换头指针指向
tags:
---

git checkout 


HEAD 可以理解为头指针，是当前工作区的基础版本，在工作区指向的提交会以头指针指向的提交为父提交。

如果想要查看 HEAD 的指向可以使用一下命令：
> cat .git/HEAD

使用 git checkout 命令检出最新提交的父提交：

> git checkout master^

当然以上命令可以将 master 换成具体 commit 的 hash 值。

查看当前 HEAD 的内容：

> cat .git/HEAD

那么此时的指针的状态就称为“分离头指针”，因为此时的 HEAD 头指针指向了一个具体的提交而不是一个引用（分支）。

通过 git reflog 可以看到当前的头指针的指向。

使用以下命令可以看到 HEAD 和 master 对应不同提交：

> git rev-parse HEAD master


针对切换 HEAD 后的工作区做一些改变，创建文件 two，提交版本库(提交信息为：create two)，编辑 two 文件，提交信息为 edit two。



重新切换回 master 分支，此时上两次在分离头指针状态下的两次提交消失了（相应的文件更改也不见了），但是还是可以通过 git reflog 查看到上两次提交的记录。


其实这两个提交还在版本库中，只是因为它们没有被任何一个分支跟踪到，所以不能保证这些提交会一直存在，实际上当 reflog 中含有的提交日志过期后，这些提交随时会被在版本库中彻底清除。


那么我们如何才能将以上更改应用到 master(任何其他) 分支呢？

我们可以使用 merge 将以上提交合并到 master 分支上。

1. 首先应该确认处于 master 分支上：

> git branch 或 git branch -v

2. 执行合并操作

> git merge xxxx

如果分支头指针后有多次提交，那么上面命令会将 xxxx 之前的提交一并合并到 mastr 分支上。


---

git checkout 的用法：


