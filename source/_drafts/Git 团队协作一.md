---
title: Git 团队协作(一):基本了解
tags:
---


### Git 的配置文件

用户主目录下的 `.gitconfig` 或系统文件 `/etc/gitconfig`，其中的配置永久生效。



以上配置文件该如何配置，配置的变量以及格式是什么？



**配置 Git 用户名和邮件地址**

> git  config --global user.name "xxx"

> git  config --global user.email "xxx"

**为 Git 设置别名**

>$ git config --global alias.st status

也可以包含命令参数：

>$ git config --global alias.ci "commit -s"

以后使用 `git st` 就相当于 `git status`。

也可使使用 sudo 命令，新建的别名可以被所以用户使用：
>$ sudo git config --global alias.st status


> git config --system alias.st status


**git config 命令的参数及区别**

在工作区目录(/a/b/demo)中执行以下命令，将打开 /a/b/demo/.git/config 文件进行编辑。

> $ git config -e

执行如下命令，将打开用户目录下的 .gitconfig 进行编辑：

> $ git config -e --gobal


执行如下命令，将打开系统 Git 配置文件进行编辑(比如：/usr/local/git/etc/gitconfig)：
> $ git config -e --system


版本库级别的配置文件、用户配置文件、系统配置文件，优先级依次下降，前者可以覆盖后者。


**编辑配置文件**

使用 3 个配置文件采用了 INI 文件格式，大致显示如下：


```
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
```
git config 命令用来查看和修改 INI 配置文件的内容。

* 读取：

>$ git config <section>.<key>

示例：

>$ git config core.filemode
true

* 修改

>$ git config <section>.<key> <value>

示例：

在项目的工作区执行以下命令：

>$ git config a.b hahah
>$ git config x.y.z zzzz

修改后的文件

```
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
[a]
	b = hahaha
[x "y"]
	z = zzzz
```

Git 配置时的变量可以查看官方文档：[git-config](https://git-scm.com/docs/git-config)

### 工作区、暂存区、Git 版本库

使用 `git init` 创建版本库，在当前目录下回生成 .git 隐藏目录。


这个 .git 目录就是 **Git 版本库**(库、repository)，而 .git 所在的目录称为 **工作区**。




### 工作区根目录下的 .git 目录

Git 将版本库放在工作区的根目录，这使得所有的版本控制操作(除与远程版本库的交互操作)都可以在本地完成。


在工作区的子目录下进行操作时，在在工作区目录中依次向上递归查找 .git 目录，.git 目录就是工作区所对应的版本库，其所在目录就是工作区的根目录，.git 下的 index 文件记录了工作区文件的状态(实际上是暂存区的拽他)

