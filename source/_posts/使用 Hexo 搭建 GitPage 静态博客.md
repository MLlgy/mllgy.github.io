---
title: 使用 Hexo 搭建 GitPage 静态博客
tags: [Hexo,GitPage]
---
## 必要工具的安装(Mac)

### nmp

[node 官方网站](https://nodejs.org/en/)下载 pkg 包，进行安装 node.js 环境，安装完毕 npm 也被安装完成。

或使用相关命令行进行安装，具体步骤自行搜索。


### hexo
```
$ npm install -g hexo-cli
```
具体参见 [Hexo 官方文档](https://hexo.io/zh-cn/docs/)

## Hexo 建站

执行以下命令：

```
$ hexo init <folder>
$ cd <folder>
$ npm install
```
编辑 `_config.yml` 文件，进行相关配置。

<!--more-->

## 文章编写

### Create a new post

``` bash
$ hexo new "My New Post"
```
### Generate static files

``` bash
$ hexo generate
```

### Run server

``` bash
$ hexo server
```

在浏览器中打开 http://localhost:4000 进行预览

## 博文部署

`_congfig.yml` 有关文件编辑如下：
```
deploy:
  type: git
  repository: https://github.com/xx/xxx.github.io.git
  branch: master
```
### 安装 hexo-deployer-git

```
$ npm install hexo-deployer-git --save
```

### 进行部署

在每次编辑文章后执行以下操作：

```
$ hexo clean //清理 public 文件夹和 database 文件
$ hexo generate // 重新生成 public 文件夹和 database 文件
$ hexo deploy // 部署到 github page 上
```

观察了以下目录文件夹，部署到 GitPage 上应该为 `hexo g` 生成的 public 文件夹下的内容。

----

**知识链接：**

  &nbsp;&nbsp;&nbsp;&nbsp;[Hexo建站](https://cherryblog.site/categories/Hexo%E5%BB%BA%E7%AB%99/)