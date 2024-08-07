---
title: 合并和拆分Git仓库
author: 张帆
tags:
  - Git
date: 2024-08-02 10:16:14
---

通常来说，一个Git仓库代表了一个项目，Git仓库会和该项目的生命周期保持一致，但在某些情况下，我们可能需要将多个代码仓库进行合并，又或者我们需要将一个仓库进行拆分。最粗暴的方式就是直接将仓库文件移动到全新仓库，但这样做会导致历史提交信息丢失，这些提交信息往往对于后续的代码阅读和问题分析是一笔宝贵的资料。我们需要寻找在不丢失提交信息的前提下的仓库合并/拆分方式。

<!--more-->

## 合并

A团队的代码仓库存在一个问题是代码模块拆分的过细，本意是为了减少模块间耦合，提高模块复用性，但随着项目的发展，模块越来越多，对应的代码仓库也越来越多，给项目管理上带来了很大的负担。同时模块之间依赖形成了一张复杂的依赖网。因此，我们需要将这些过细的模块进行合并，减少模块间依赖，简化项目管理。这就需要用到代码仓库合并。下文以合并RepoA/RepoB/RepoC/RepoD这四个功能上类似的库为RepoE为例，具体步骤如下：

1. 创建新的空仓库RepoE并克隆到本地
2. 将RepoA仓库添加为远程仓库，命名为RepoA，并同步代码
    - `git remote add --fetch RepoA git@gitxxx.com/RepoA.git`
3. 将RepoA代码合并至主分支
    - `git merge RepoA/master --allow-unrelated-histories`
4. 创建临时目录RepoA并将所有文件移动到该目录，然后创建提交。注意这里不能使用`*`匹配，会导致`.git`文件夹也被移到到子目录。该步骤主要目的是为了避免后续合并其他仓库代码时发生冲突
    - `mkdir RepoA && git mv <files> RepoA`
    - `git add . && git commit -m "添加RepoA源码"`
5. 重复上述2，3，4步，将RepoB/RepoC/RepoD源码同步至对应文件夹

这样，我们就完成了将多个仓库在保留提交信息的前提下的合并。接下来的工作主要有

- 整合通用配置文件，如.gitignore等，保留一份或合并内容即可；
- 整理目录结构，合并CMake工程。这里建议保留原项目目录结构，并基于CMake Interface将多个项目合并，这样改动最小


## 拆分

某B应用是一个双进程项目，UI和具体业务逻辑分离在两个进程，方便后续更换UI框架。原计划是将后台业务/应用进程代码放在一个仓库，作为一个CMake工程管理，减少依赖层级，但实际开发中发现该结构使得调试较为麻烦，难以同时调试后台进程和UI应用，也使得本应该没有任何直接代码依赖的UI应用和后台进程容易在开发中引入一些代码层面上的依赖，这对于后续更换UI框架/语言而言是个不好的苗头。因此需要将这个仓库拆分为两个，新建一个仓库用于开发业务后台进程，原仓库用于UI应用。具体步骤如下：

1. 安装git-filter-repo工具
    - `pip install git-filter-repo`
2. 将原仓库RepoFull克隆到本地，指定文件夹名为RepoDaemon
3. 将需要保留的文件/文件夹相对于项目根目录的路径写入/tmp/keep.txt文件中。注意：部分每次提交都会修改的文件建议不添加，等所有操作完成后再添加回去，这类文件的修改信息往往不是那么重要，反而会导致出现大量仅对该文件有修改的无意义提交。
4. 执行过滤操作，仅保留对上一步指定的文件/文件夹有操作的提交
    - `git filter-repo --paths-from-file /tmp/keep.txt --force`
5. 将原仓库从远程仓库中移除，并将新仓库添加为origin远程仓库，并关联分支及推送
    - `git remote rm origin`
    - `git remote add origin git@gitxxx.com/RepoDaemon.git`
    - `git branch --set-upstream-to=origin/main main`
    - `git push`
6. 在原仓库中，执行3/4两步，保留UI应用相关文件及提交
7. 创建新分支，并推送至远端，确保CI通过。
8. 将本地代码强制提交至远程仓库主分支，必须和项目成员沟通一致通过后进行（*危险！*)
