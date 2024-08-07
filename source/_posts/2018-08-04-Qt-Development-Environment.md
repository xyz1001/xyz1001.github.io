---
title: Qt从0到1之工具篇：开发环境
author: 张帆
tags:
  - Qt
  - Qt工具
  - Qt从0到1
abbrlink: 8517
date: 2018-08-04 14:42:40
---

由于Qt本身的复杂性，且有一套独特的构建工具qmake，因此并不能像其他C/C++库一样，可以很简单的配置使用。对此，Qt推出了Qt Creator这一对Qt开发有着优秀支持的IDE，简化使用难度。除了Qt Creator，我们也可以借助Visual Studio的插件Qt Visual Studio Toos插件实现对qmake项目的支持。qmake还为我们提供了qmake项目生成VS，XCode对应工程的功能。当然，Vim经过配置，也可以很舒服地进行Qt开发。

<!--more-->

## Qt Creator

Qt Creator是Qt在2008年推出的一款跨平台的IDE，可运行在Windows，Linux，MacOS等平台，提供了对qmake和cmake项目的良好支持，可以很好地支持C++及QML开发。Qt Creator不仅仅可以用于开发Qt项目，完全可以作为一个C++ IDE使用。对于Qt Creator的配置和使用官方文档有很详细的说明，网上也有相当多的文章，这里不多赘述，仅挑几点我常用的进行说明。

### 构建套件配置

Qt Creator的构建套件是指用于Qt开发所需要的一系列外部工具的集合，包括了Qt SDK，编译器，调试器等，相关设置可在选项(Options)->套件(Kits)中进行配置，一般Qt均可以自动识别并配置构建套件，但某些特殊情况下需要手动配置，如静态编译的Qt SDK，独立安装的编译器等。

1. 配置Qt SDK时，只需要制定SDK中qmake路径即可。
2. 配置编译器时，需要注意的是构建套件中选择的编译器需要和Qt SDK对应。如Qt SDK是VS2017 64位的，编译器也需要为VS2017中提供的64位编译器，而不能使用32位或MingGw编译器。这里我遇到的一个特殊情况是由于我需要使用32位的VS2017编译程序，但Qt默认仅提供了VS2015 32位编译器预编译的SDK，由于VS2017和VS2015二进制兼容，所以我们可以选择VS2015 32位的Qt SDK + VS2017 32位编译器。不过Qt默认是完全匹配才会自动配置，所以在安装完后需要手动选择编译器。
3. Qt默认在Windows下且VS作为编译器时使用cdb作为调试器，但由于Windows一般不会安装cdb，所以我们需要手动安装。cdb目前已不提供独立的安装包，微软提供了几种[安装方式](https://docs.microsoft.com/zh-cn/windows-hardware/drivers/debugger/debugger-download-tools)，常用的方式为通过WDK进行安装。我们可以将CDB离线下载下来，方便以后使用。

### 插件支持

Qt Creator默认集成了一些插件，可在帮助->关于插件中启用或关闭。列举一些我常用的插件。

1. FakeVim
 该插件提供了对Vim编辑模式的支持，而且可以支持导入Vim基本配置文件，对于Vimer来说必不可少。当然，Qt Creator提供了EmacsKeys插件，Emacs用户可以尝试一下。
2. Beautifier
 该插件用于代码的格式化，其提供了包括`clang-format`，`astyle`在内的代码格式化工具的支持。
3. Valgrind
 这个插件提供了对`Valgrind`的支持，默认是开启的。`Valgrind`是一个用于内存调试，内存泄漏检测和性能分析的命令行工具。我们可以在Qt Creator的分析菜单中找到`Valgrind`相关的功能项，Qt Creator会将相关结果在图形化界面上展现出来。需要注意的是在使用前我们需要先安装`Valgrind`这个软件。

## Visual Studio

VS是一款强大的IDE，其提供如远程调试，性能分析等其他IDE或编辑器难以提供的工具。我们可以通过以下两种方式来使用它来进行Qt开发。

### Qt Visual Studio Tools

`Qt Visual Studio Tools`是一个Qt官方开发的VS插件，我们安装安装完成后配置下Qt SDK安装目录即可。具体可参考[使用文档](http://doc.qt.io/qtvstools/qtvstools-getting-started.html)

### qmake生成VS工程

qmake支持生成VS工程，执行命令`qmake . -t vcapp`即可。

## Vim

对于一个Vimer，即使Qt Creator和VS提供了Vim模式，但还是习惯使用Vim来编写代码。使用Vim的最大问题便是补全，目前我使用`Valloric/YouCompleteMe`来进行补全，并使用`rdnetto/YCM-Generator`来生成配置文件，`YCM-Generator`提供了对qmake工程的支持。为了能方便的使用Vim进行Qt开发，我们还需要熟练掌握Qt 命令行工具，如`rcc`，`qmake`，`lupdate`等的使用。


