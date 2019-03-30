---
title: 踩坑记之Jenkins无权限克隆的代码
author: 张帆
tags:
  - 踩坑记
  - Jenkins
  - 运维
date: 2019-03-25 18:06:57
---

## 踩坑

最近部门新采购了一台Mac Mini用作Jenkins自动构建服务器，由于之前搭建过Windows构建服务器，因此这次的Mac构建服务器的任务自然落在我的身上。开始一切都非常顺利，安装好必须的软件，配置好环境，但在正式上线，构建一个跨平台项目时，却发现macOS服务器始终无法从公司自建的Gitlab代码托管平台上克隆代码，而同一项目的其他平台（Windows和Linux）服务器却一切正常。macOS服务器的出错提示信息如下。

```
> git rev-parse --is-inside-work-tree # timeout=10
Setting origin to gitlab.gz.cvte.cn:zhangfan3427/ffmpeg-conan.git
> git config remote.origin.url gitlab.gz.cvte.cn:zhangfan3427/ffmpeg-conan.git # timeout=10
Fetching origin...
Fetching upstream changes from origin
> git --version # timeout=10
using GIT_SSH to set credentials mh1602 private ssh key for git submodule
> git fetch --tags --progress origin +refs/heads/*:refs/remotes/origin/*
hudson.plugins.git.GitException: Command "git fetch --tags --progress origin +refs/heads/*:refs/remotes/origin/*" returned status code 128:
stdout:
stderr: Permission denied (publickey).
fatal: Could not read from remote repository.
Please make sure you have the correct access rights
```

由于这个项目是多个平台同时构建，各个平台的Jenkins项目配置是完全一致的，因此可以排除掉Jenkins配置问题，难道是macOS服务器配置错了？

<!--more-->

## 出坑

### SSH密钥错误？

刚遇到这个问题，我第一反应是不是macOS构建服务器上的SSH密钥是不是配置错了，但很快就排除了这个想法，因为之前在搭建Windows构建服务器时也出现过一模一样的现象。当时利用procmon监控jenkins slave进程，发现jenkins的在克隆代码过程中工作流程大致如下：




