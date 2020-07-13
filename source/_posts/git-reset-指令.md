---
title: git reset 指令
date: 2020-06-26 17:14:18
tags:
---


**git reset 使用场景为丢弃 commit 后的 commit 信息、index 信息或者源码。**

# git reset 三种模式

* soft
* mixed(默认)
* hard



## 1. git reset --soft HEAD~n 或 commit id  

*回退项：* commit信息

*回退情况：* 当前 commit 与目标 commit 信息之间的 **commit 信息丢失**，此时index信息未发生改变，执行 `git commit` 信息，此时相当于 **将丢失的 commit 信息合并为新的 commit 信息**。


## 2. git reset （--mixed）HEAD~n 或 commit id 
*回退项：* index 信息、commit 信息

*回退情况：* 当前 commit 与目标 commit 信息之间的 commit 信息丢失，此时 index 信息也发生改变---变为 
 **unstage** 状态,即丢失的 commit 信息对应的源码变为 unstage 状态，需执行 `git add  . ` 将文件变更暂存，执行 `git commit` 将本次操作提交到本地分支,那么此次 commit 内容包括所有丢失的 commit 内容的总和。

## 3. git reset --hard  HEAD~n 或 commit id
*回退项：* index 信息、commit信息、源码

*回退情况：* 当前 commit 与目标 commit 信息之间的 commit信息、index 信息、源码全部丢失。


 