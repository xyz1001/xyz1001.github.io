---
title: 踩坑记之Jenkins无权限克隆的代码
author: 张帆
tags:
  - 踩坑记
  - Jenkins
  - 运维
abbrlink: 35120
date: 2019-03-25 18:06:57
---

## 踩坑

最近部门新采购了一台Mac Mini用作Jenkins自动构建服务器，由于之前搭建过Windows构建服务器，因此这次的Mac构建服务器的任务自然落在我的身上。开始一切都非常顺利，安装好必须的软件，配置好环境，但在正式上线，构建一个跨平台项目时，却发现macOS服务器始终无法从公司自建的Gitlab代码托管平台上克隆代码，而同一项目的其他平台（Windows和Linux）服务器却一切正常。macOS服务器的出错提示信息如下(部分信息用*代替)。

```
> git rev-parse --is-inside-work-tree # timeout=10
Setting origin to xxx.com:user/repo.git
> git config remote.origin.url xxx.com:path/repo.git # timeout=10
Fetching origin...
Fetching upstream changes from origin
> git --version # timeout=10
using GIT_SSH to set credentials **** private ssh key for git submodule
> git fetch --tags --progress origin +refs/heads/*:refs/remotes/origin/*
hudson.plugins.git.GitException: Command "git fetch --tags --progress origin +refs/heads/*:refs/remotes/origin/*" returned status code 128:
stdout:
stderr: Permission denied (publickey).
fatal: Could not read from remote repository.
Please make sure you have the correct access rights
```

由于这个项目是多个平台同时构建，各个平台的`Jenkins`项目配置是完全一致的，因此可以排除掉Jenkins配置问题，难道是macOS服务器配置错了？

<!--more-->

## 出坑

在继续说明问题之前，我需要先简单介绍一下`Jenkins`的在克隆代码步骤（即`pipeline`中的`checkout scm`命令）的过程。

### Jenkins克隆代码流程

`Jenkins`的在克隆代码过程中工作流程大致如下：

![Jenkins克隆代码](jenkins_clode_code.png)

1. 当我们触发`Jenkins`构建时，`Jenkins`主节点收到构建请求。在我们的项目的构建配置信息中会指定构建的子节点名和构建步骤，主节点会根据配置信息将具体的任务分派给指定的子节点。一个项目可以指定多个子节点进行不同的构建任务。
2. 子节点在收到构建任务后，如果存在`Git`克隆代码的任务，子节点会向主节点请求克隆代码所需的账号密码或`SSH`密钥，`jenkins`称之为`credentials`（凭证）。这样做的好处在于子节点无需部署证书信息，管理员只需要将敏感的证书信息保存在主节点并管理好主节点的权限，就可以保证证书的保密性。
3. 对于不同类型的`credentials`，`Jenkins`会做不同的处理。对于账号密码类型的`credentials`，我没有深入了解处理方式，本文所处理的是`credentials`为`SSH`密钥的情况。
4. 子节点在获取到主节点发来的`SSH`密钥后，会将`SSH`密钥保存在一个临时目录。
5. 我们知道`Git`在使用`SSH`协议克隆代码时，会调用`SSH`来连接代码仓库服务器，我们最常见的`Git`仓库地址，类似于`git@xxx.com:user/repo.git`(例如`git@github.com:torvalds/linux.git`)，其实是一个`SSH`链接，其中用户名为`git`。然后`SSH`会默认使用`$HOME/.ssh/id_rsa`文件的内容作为`SSH`密钥，很多`Git`的配置教程第一步就是生成该文件。`SSH`也可以通过`-i identify_file`参数来手动指定一个文件作为`SSH`密钥文件。相应的，`Git`也提供了一种方式让我们可以配置其调用`SSH`的参数，这种方式就是上面出错提示信息中的`GIT_SSH`。这是一个环境变量，`Git`官方文档中对其的说明如下：

 > GIT_SSH, if specified, is a program that is invoked instead of ssh when Git tries to connect to an SSH host. It is invoked like $GIT_SSH [username@]host [-p <port>] <command>. Note that this isn’t the easiest way to customize how ssh is invoked; it won’t support extra command-line parameters, so you’d have to write a wrapper script and set GIT_SSH to point to it. It’s probably easier just to use the ~/.ssh/config file for that.

 简单来说，就是`GIT_SSH`用于指定一个程序的路径，如果用户设置了该环境变量，`Git`就会使用指定的程序来代替默认的`ssh`命令，即使用`$GIT_SSH [username@]host [-p <port>] <command>`的形式去连接`SSH`服务器。
 在这一步中，`Jenkins`在Unix环境下会创建一个`shell`脚本，脚本中的路径类似`ssh -i <SSH密钥路径> $*`，并设置环境变量`GIT_SSH`为该脚本的路径。
6. `Jenkins`使用刚创建的`shell`脚本去连接`Git`服务器
7. 克隆代码
8. 克隆代码完成后，为了安全性考虑，`Jenkins`会立即将刚创建的两个文件，`SSH`密钥文件和`GIT_SSH`指定的`shell`脚本删除。

### 失败的尝试

- `SSH`密钥错误？
 刚遇到这个问题，我第一反应是不是macOS构建服务器上的`SSH`密钥是不是配置错了，但很快就排除了这个想法。因为根据上述`Jenkins`克隆代码的操作中我们可以看到，整个过程中都不会去取`$HOME/.ssh/id_rsa`文件，因此服务器上配置的`SSH`密钥对于`Jenkins`没有任何影响。
- `Git`版本不对？
 然后又想到之前在搭建Windows构建服务器时也出现过一模一样的现象。当时的处理方式是把`Git`版本从最新的2.19降到了2.13就可以了，当时的猜测是可能最新的`Git`和`Jenkins`不兼容导致。于是满怀希望的将macOS服务器上的`Git`版本从自带的`2.17`降级到`2.13`，然而发现并没有任何用处。又尝试了多个版本的`Git`，均以失败告终。
- 不支持`GIT_SSH`？
 我接着尝试仿照`Jenkins`克隆代码的流程，发现可以正常克隆代码，说明`Jenkins`的流程是没有问题。

后续又尝试了其他可能的原因，也寻求了其他同事的协助，但都没有找到问题的根源。网上相关的资料也都没有找到可用的方案。就这样折腾了一天，我陷入了深深地绝望之中。

### 转机

就在我一筹莫展时，一位同事提示说我们更透彻的了解`Jenkins`克隆代码的流程，虽然现在我们知道一个大致的流程，但却对细节并不了解，我们需要拿到`GIT_SSH`指定的`shell`脚本的内容。但如何拿到这个临时脚本的内容呢？这个临时脚本的只存在于`Jenkins`执行`checkout scm`这条命令期间，我们好像没有任何办法插入自己的命令去获取到这个脚本。

### 偷梁换柱

如果我们把`git`命令给替换掉呢？相信很多Unix用户都会有这样的经验，就是用一个命令覆盖掉旧的命令。如在安装了`oh-my-zsh`后，真正的`ls`命令会被替换为`ls --color=tty`用于支持颜色，我们执行`ls`时实际上执行的是`ls --color=tty`，这是通过`alais`来实现的。我们也可以通过直接将原程序直接替换来实现这样的效果。有了这个思路后，获取`GIT_SSH`指定的`shell`脚本的内容就变得很简单了。

1. 首先我们将真的`git`重命名为`git_bak`，然后创建一个名为`git`的`shell`脚本，输入以下内容

 ``` bash
 #!/bin/bash

 env >> ~/env.log # 将环境变量保存至文件
 /usr/local/git/bin/git_bak $*
 ```

2. 重新点击构建，触发`Jenkins`克隆代码流程。果然，`Jenkins`使用了假的`git`命令，我们成功获取到了`Jenkins`执行`checkout scm`的环境变量。其中比较重要的几个环境变量如下：

 ``` bash
 GIT_SSH=/path/to/jenkins/workspace/project_build_dir_name@tmp/ssh3913422597709085290.sh
 PWD=/path/to/jenkins/workspace/project_build_dir_name
 ```

 我们可以看到，`Jenkins`使用的保存`SSH`密钥的临时目录就是项目构建目录+`@tmp`。如果有进入过`Jenkins`的工作区目录看过可以发现，所有的项目构建都会产生两个文件夹，一个就是我们在项目构建页面上看到的工作区，真正的项目构建目录，里面包含了构建的文件，产物等，而另一个文件夹命名为`前一个文件夹名+@tmp`，点进去看往往是空的。原来这个目录就是用于存在构建过程中的需要及时删除的临时文件。

3. 知道了上面这个信息，我们在对假的`git`脚本稍作修改，就可以把`Jenkins`生成的临时文件都复制出来了。

 ``` bash
 #!/bin/bash

 env >> ~/env.log # 将环境变量保存至文件
 cp ${PWD}@tmp/* ~ # 将临时目录下的文件复制出来

 /usr/bin/git_bak $*
 ```

 再次重新构建，我们就拿到了`Jenkins`生成的临时文件，一共包括两个文件，一个是`GIT_SSH`指定的脚本，另一个就是`SSH`密钥了。其中`SSH`密钥校验后并没有问题，那问题应该就出在脚本上了。

4. 脚本文件的内容如下

 ``` bash
 #!/bin/sh
 if [ -z "${DISPLAY}" ]; then
     DISPLAY=:123.456
     export DISPLAY
 fi
 ssh -i "/path/to/jenkins/workspace/project_build_dir_name@tmp/ssh853779284307646112.key" -l "jenkins_user" -o StrictHostKeyChecking=no "$@"
 ```

 最重要的部分就是脚本中的最后一行，和我仿照`Jenkins`克隆代码流程中创建的脚本不同的是其还添加了两个参数`-l "jenkins_user" -o StrictHostKeyChecking=no`。后一个参数用于禁用`SSH`公钥检查，理论上不会影响`SSH`的认证。而前一个参数`-l "jenkins_user"`就很奇怪了，`jenkins_user`是在`Jenkins`上添加`credentials`时输入的用户名，这个参数的作用是用于指定`SSH`登录用户名。上面提到，在使用`SSH`的方式克隆代码时，`SSH`登录用户名已经在连接中指定为`git`。这样在两个地方都指定了`SSH`的登录用户名，应该会产生冲突。出问题的MacOS服务器上应该是使用了`-l`参数指定的用户名，导致无法登录。那为什么旧的MacOS服务器可以正常克隆代码呢？我对比了两台服务器的`OpenSSH`版本，旧的MacOS服务器上`OpenSSH`版本较旧，猜测`OpenSSH`使用哪个作为认证时的用户名是和其版本有关，旧版本的`OpenSSH`使用链接中的用户名，而新版本的`OpenSSH`则修改为使用`-l`参数指定的用户名。这也可以解释为什么Windows服务器上降级了`git`版本就可以正常工作了，因为在Windows上，`OpenSSH`是集成在`git`安装包中的，安装旧版本的`git`安装包后，其实也把`OpenSSH`的版本降低了，阴差阳错间避开了这个问题。

## 填坑

至此，问题的根源终于找到，要解决问题有两种方式，一种统一两处指定的用户名，即修改`Jenkins`上添加的`credentials`信息，将用户名修改为`git`，另一种是将服务器上的`OpenSSH`版本降级到旧版本。这里我选用的是第一种方式，便于后续添加新的构建服务器时不再次踩坑。

## 总结

1. 在整个过程中，对`Jenkins`克隆代码的流程有一个基本了解给解决这个问题提供了很大帮助。上述对`Jenkins`克隆代码步骤的探索就是当时在处理Windows服务器上无法克隆代码时通过[Procmon](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon)工具观察得到。可惜当时降级`git`版本后问题没有复现就没有继续深究，导致又一次踩坑。
2. 可以看到，`GIT_SSH`指定的其实就是一个“假”的`ssh`命令，和我们解决问题中使用的“假”的`git`命令的有异曲同工之妙。
3. 在使用“假”的`git`命令时，我们还拿到了十分敏感的`SSH`密钥，这说明了`Jenkins`的这项保密策略其实是有很大的漏洞的。只要子节点服务器管理员拥有root权限，就可以十分轻松的获取到管理员保存在主节点上的密钥，进而窃取其他代码仓库中的代码。为了避免这个问题，管理员需要严格控制子节点的root权限和敏感操作，不同小组的代码最好能够添加不同的`credentials`进行拉取。
