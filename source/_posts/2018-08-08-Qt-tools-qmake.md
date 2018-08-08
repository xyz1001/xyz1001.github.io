---
title: Qt从0到1之工具篇（2）：qmake
author: 张帆
tags:
  - Qt
  - qmake
date: 2018-08-08 10:59:16
---

## 简介

和cmake类似，qmake也是一个跨平台的构建工具，目前主要用于Qt项目的构建。本文主要基于官方文档和个人的经验做了一些介绍，更加详细的内容可以参考[官方手册](http://doc.qt.io/qt-5/qmake-manual.html).
相对于其他的构建工具，qmake有一些特性：

- 跨平台支持。为了满足Qt的跨平台性，qmake也支持运行在目前绝大部分操作系统，如Windows，Linux和MacOS，Android等。
- 支持生成主流IDE的工程文件。qmake可以生成`Visual Studio`和`Xcode`的工程项目文件，便于使用IDE提供的工具。
- 对Qt项目的完美支持。对于Qt项目，qmake可以帮我们自动配置相关编译流程，如MOC预处理，库的链接等。
- Qt Creator提供了对qmake的良好支持。如果我们使用Qt Creator + qmake来构建一个项目，我们几乎不需要手动去修改qmake的配置文件，Qt Creator可以自动帮我们配置。

目前我们的Qt项目都是使用qmake进行构建管理，而库相关的项目主要使用cmake进行构建管理。执行qmake命令后，qmake会寻找制定路径下的`pro`文件并生成对应平台的`makefile`文件，然后我们就可以使用该平台的`make`工具进行编译。

<!--more-->

## 语法

qmake的语法相对来说比较简单，和cmake相比，关键词要少很多，Qt官方手册里面有一章详细的介绍了所有关键词的使用。这里也推荐一篇博客，里面对qmake的关键词有比较详细的讲解。
- [Qt Variables](http://doc.qt.io/qt-5/qmake-variable-reference.html)
- [Qt之pro配置详解](https://blog.csdn.net/liang19890820/article/details/51774724)

这里列举一些qmake使用过程中需要注意的地方或小技巧。
- 将环境变量转换为宏定义。在软件的自动构建中，我们可能会需要根据环境变量来确定编译期的一些行为或数据，如我们可能在环境变量中设置了当前软件的版本号，我们需要将这个版本号设置到我们的代码中。这个时候我们可以通过读取环境变量来设置一个宏，这样我们就可以在代码中使用。对于qmake，我们可以通过下面这行语句来设置。
```
DEFINES += [micro]=\\\"$$([environment variable])\\\"
```
 其中`[environment variable]`是我们要读取的环境变量，`[micro]`是我们要设置的宏
- qmake目前不支持相同文件名的源文件。即在当前项目中不能存在两个一样名称的`.cpp`文件，即使是在不同目录下。因为qmake默认编译输出的目标文件（`.o`或`.obj`文件）放在同一目录下，两个同名的`.cpp`文件生成的目标文件是也是同名的，后一个生成的会覆盖前一个生成的。这是qmake的一个缺陷，cmake不存在这样的问题。
- 我们可以添加`QMAKE_PROJECT_DEPTH = 0`使qmake在生成`makefile`时使用绝对路径。这一点在某些情况下会非常有用。例如如果使用Vim开发，qmake编译生成的输出在quickfix窗口中的路径是相对路径，回车无法跳转到正确文件，设置使用绝对路径就可以解决这个问题。


## 使用

我们执行`qmake -h`就可以看到qmake的基本使用说明。其用法为

> qmake [mode] [options] [files]

其中mode有两种，一种是生成工程文件，通过`-project`参数制定，另一种是生成`makefile`，通过`-makefile`参数指定。若用户未指定，则默认使用第二种模式。

1. 生成工程文件
这种模式一般用于直接在命令行中初始化一个qmake项目。qmake会在当前目录生成一个`pro`文件，寻找指定目录下所有相关文件添加到相应的参数中。

2. 生成makefile
这是我们最常用的模式。我们需要制定`pro`文件的路径或目录，用于生成`makefile`文件。若目录下仅有一个`pro`文件，我们可以直接制定路径即可，qmake会自动寻找`pro`文件，否则我们需要制定具体的`pro`文件路径。
qmake支持`shadow build`，也叫`out of source build`，就是将源码路径和构建路径分开（也就是生成的makefile文件和其他产物都不放到源码路径），以此来保证源码路径的清洁。我们一般会在源码目录建立一个`build`目录，在该目录执行`qmake ..`命令。这样我们的构建文件都在`build`目录，需要清理时直接删除`build`目录即可。
在生成makefile之后，我们可以调用平台提供的make工具，如类UNIX系统上，我们可以直接执行`make`命令，而Windows上，我们可以执行`nmake`命令进行编译。
编译完成后，若我们在`pro`文件中制定了需要安装的文件，我们可以执行`make install`或`nmake install`来进行文件归档。
