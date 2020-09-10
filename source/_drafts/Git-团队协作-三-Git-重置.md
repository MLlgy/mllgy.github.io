---
title: 'Git 团队协作(三):Git 重置'
tags:
---


master 分支在版本库的引用目录(.git/refs)中体现为一个引用文件： .git/refs/head/master，其文件内容就是分支中最新提交的提交 ID。




引用 ref/heads/master 就好像一个游标，在新的提交发生时指向新的提交，
既然是游标，那么就可以做到可上可下，Git 提供 git reset 命令，可以将游标指向任意一个存在的提交 ID。

```
git reset --hard commitId： 使用 commitid 对应的版本库内容覆盖暂存区和工作区。
git reset --mixed commitId(默认)： 暂存区 恢复到 commitid 对应的版本库，工作区不变。
git reset --soft commitId：恢复到对应的 commitID 的版本库，但是不会改变暂存区和工作区(将版本库恢复到提交指定commitId前的状态)。
```


### 使用 relog 挽救错误的重置

