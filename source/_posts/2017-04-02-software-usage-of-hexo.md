---
title: 软件折腾笔记之hexo
author: 张帆
tags:
  - 软件折腾笔记
  - hexo
date: 2017-04-02 17:05:47
---


hexo是一个基于nodejs、轻量、原生支持markdown写作的的博客系统。在看了一位同事的博客之后，我坚定地转向了hexo。但在安装配置和使用的过程中摸索了较长时间，也遇到了较多的坑点。 如果是初次接触hexo，推荐去阅读一下hexo的[官方文档](https://hexo.io/zh-cn/docs/)，会让我们对hexo的使用有一个基本的了解。 在搭建自己的hexo博客的过程中，我较多的参考了这位同事对hexo介绍的一篇[文章](http://blog.guorongfei.com/2016/01/01/update-blog-with-hexo/)及他托管在Github上的[hexo博客](https://github.com/zhaohuaxishi/zhaohuaxishi.github.io)。

<!--more-->

## 前提

 - debian系列Linux系统(我用的是Deepin)
 - 使用Github Pages搭建博客
 - Git已配置好ssh key并已部署公钥到Github

所需要的相关软件安装命令如下：
``` bash
sudo apt install git npm nodejs-legacy #安装git
sudo npm install npm@lts -g
sudo npm install -g hexo-cli    #安装hexo命令行接口，可通过命令使用hexo
```
其他情况请自行作出修改。

## hexo搭建

### 操作步骤

通常情况下的做法是在Github上建立一个Github Pages仓库`yourname.github.io`用来存放生成静态网页供访问，另外再新建一个仓库存放hexo及markdown源码，此类情况下的hexo博客搭建教程网上已经有很多，在此不再赘述。但由于这样做导致需要两个管理两个仓库，相对比较麻烦。因此我参考了同事的做法，使用不同的分支进行管理。具体方法如下：

1. 创建Github Pages仓库；
2. `git clone`至本地；
3. 创建`.gitignore`文件，文件内容如下：
``` bash
*
!.gitignore
```
即忽略除了`.gitignore`以外的所有文件，然后添加后提交；

4. 创建新分支`source`，切换至`source`；
5. 在外部新建一个临时文件夹`hexo_init`并在此目录初始化`hexo`；
6. 将`hexo_init`目录中所有文件复制至仓库文件夹并删除临时文件夹`hexo_init`；
7. 修改`hexo`配置文件`_config.yml`，将`deploy`条目做如下修改，注意添加上你的`Github Pages`仓库地址：
```
# Deploymen
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: #your github pages repo
  branch: master
  message: 
```
8. 推送`source`分支至Github
9. 在Github上将`source`分支设为默认分支

### 部分命令

``` bash
git clone #your github pages repo    #克隆Github Pages仓库
echo "*\n\!.gitignore" > .gitignore   #创建本地仓库master分支下.gitignore
git add . && git commit -m "忽略本地master分支所有文件"
git checkout -b source      #创建并切换至新分支sourece
cd .. && mkdir hexo_init    #创建初始化hexo的临时目录
cd hexo_init && hexo init   #初始化hexo
cd .. && mv hexo_init/* hexo_init/.gitignore #your repo dir    #复制文件至你的仓库目录下
rmdir hexo_init    #删除临时文件夹
git add . && git commit -m "初始化hexo"    #添加并提交source分支更改
git push --set-upstream origin source   #推送新创建的分支source至Github
```

### 相关问题

- **为什么要忽略本地master分支所有文件？**

> 因为在本地仓库，我们是用不到master分支中的任何文件的。master分支后续是由`hexo-deploy-git`插件控制，我们并不直接使用，因此我们忽略本地仓库master分支内所有文件，以免后续误操作。

- **为什么不在仓库目录直接初始化hexo?**

> 因为这样会删除掉你的`.git`文件导致仓库被破坏！`hexo init`内部实现会先清空当前目录`git`相关信息

- **为什么要将`source`分支设为默认？**

> 博客搭建完成后一般都是在`source`分支下写作，极少会切换至`master`分支

## 主题管理

主题位于`hexo`目录`theme`目录下，由于往往需要对主题进行自定义，主题也需要进行同步。默认情况下，主题是一个单独的仓库，因此需要将主题文件夹中的`.git`信息删除。

## hexo恢复

当更换电脑或本地文件丢失时，我们需要在新电脑上恢复hexo，具体步骤如下：
1. 安装`git`和`npm`,`hexo-cli`
```
sudo npm install hexo-cli -g
```
2. 克隆博客仓库至本地
3. 安装npm模块
```
npm install
```
