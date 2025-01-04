---
title: Git命令整理
author: 张帆
tags:
  - Git
date: 2017-12-19 21:52:46
---

## 简介

本文用来记录日常Git使用过程中接触到的一些略高级的Git命令用法，这些命令通常是用于解决特定场景下Git相关的特定问题，以备日后查询使用。
大部分命令来源于网络，这些命令均会添加原网址，不保证链接的有效性。

记录格式为

> ### [标题，通常是使用场景的概括]
 - 使用场景：[该命令用于解决的问题]
 - 命令：[Git命令]
 - 说明：[命令解释]
 - 注意：[使用时的注意事项]
 - 引用：[命令的引用网址]

<!--more-->

## 命令

### 删除已合并的分支

- 使用场景：当前仓库存在多个已合并的无用分支，需要一次性删除
- 命令：`git branch --merged | egrep -v "(^\*|master|develop)" | xargs git branch -d`
- 说明：该命令首先通过`git branch --merged`列出所有已合并的分支，然后从中排除`master`和`develop`分支，因为这两个分支往往是重要分支，不能删除，最后将要过滤后的分支进行删除。
- 注意：如果还有其它的重要分支，需要在过滤列表(`括号中`)中添加并通过`|`分隔，推荐删除前先执行第一个命令确认要删除的分支。
- 引用：`https://stackoverflow.com/questions/6127328/how-can-i-delete-all-git-branches-which-have-been-merged`

### 忽略已跟踪的文件

- 使用场景：某些文件因为某些原因已被提交到Git仓库，现在希望忽略这些文件，但更新`.gitignore`后无效
- 命令：`git rm -r --cached . && git add . && git commit -m "update .gitignore"`
- 说明：`.gitignore`只能忽略那些原来没有被跟踪的文件，如果某些文件已经被纳入了版本管理中，则修改`.gitignore`是无效的。该命令首先删除本地缓存，使所有文件变成`untrack`状态，然后再添加并提交。
- 注意： 若仅想忽略少量几个文件，可直接执行`git rm -r --cached + 文件名（支持通配符）`，然后再添加并提交。当前仓库文件数量较多时，这样会快很多
- 引用：`http://www.pfeng.org/archives/840`

### 添加代理

- 使用场景：通过代理访问远程仓库
- 命令：`git config --global http.proxy http://127.0.0.1:1080 && git config --global https.proxy https://127.0.0.1:1080`
- 说明：为Git设置Http和Https代理，此命令将会向`~/.gitconfig`中写入配置信息
- 注意：仅对通过http和https访问的远程仓库有效，对ssh访问的远程仓库无效
