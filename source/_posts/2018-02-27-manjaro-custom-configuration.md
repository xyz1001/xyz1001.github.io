---
title: Manjaro个性化配置
author: 张帆
tags:
  - Manjaro
date: 2018-02-27 21:32:16
---

使用了Manjaro有一段时间了，从最初的Ubuntu kylin，到Ubuntu，CentOS，Deepin，直到Manjaro这个发行版，被其相对简单的安装和强大的软件源所深深吸引。刚从Debian系切换过来还不是很习惯，相对于Deepin的开箱即用，Manjaro还是有一些需要手动去配置的地方。本文将以刚安装完成的Manjaro为前提，以作备忘

<!--more-->

## 基础软件安装

### 调整软件源

1. 根据速度切换软件源

 ``` bash
 sudo pacman-mirrors -i -c China -b stable
 ```

2. 向`/etc/pacman.conf`添加archlinux源

 ```
 [archlinuxcn]
 Server = http://mirrors.ustc.edu.cn/archlinuxcn/$arch
 ```

### 恢复官方软件

1. 更新软件

 ``` bash
 sudo pacman -Syy
 sudo pacman -S gnupg archlinux-keyring archlinuxcn-keyring manjaro-keyring --needed
 sudo pacman -Syu
 ```

2. 安装备份的官方软件仓库中的软件，即可通过`pacman`进行安装的软件，生成已安装软件列表的方法可参考[Manjaro下备份已安装软件包](/2018/03/02/manjaro-backup-installed-packages/)

 ``` bash
 sudo pacman -S $(< pacman.lst) --needed
 ```

## 基本环境配置

### 基于dotfiles恢复配置文件

## 软件配置

### zsh

1. 将`zsh`设置为默认shell

 ``` bash
 chsh -s /usr/bin/zsh
 ```

2. 运行zsh，将自动安装相关插件

### tmux

1. 安装tmux插件管理器tpm

 ``` bash
 git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
 ```

2. 运行tmux，按下快捷键`Ctrl+a`，`I`安装tmux插件

### vim

1. 运行vim，自动安装插件
