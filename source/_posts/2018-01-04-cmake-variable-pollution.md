---
title: CMake"变量污染"
author: 张帆
tags:
  - CMake
date: 2018-01-04 22:40:34
---

最近在一个使用CMake进行构建的项目中，我希望在用户编译时如果通过`-D CMAKE_INSTALL_PREFIX=<path>`指定了安装目录，则使用用户指定的路径，否则为了避免默认安装到系统目录（如Linux下默认安装到`/usr/local/`目录），则另外设置一个默认安装目录，因为系统目录往往需要一定权限才能安装，，而且这个项目本身并没有安装到系统目录的需求。

查阅CMake手册我们可以知道，官方提供了`CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT`这个变量来判断`CMAKE_INSTALL_PREFIX`是否被用户手动指定。然而在使用过程中，发现这个变量经常会莫名其妙的失效。

经过查找资料和不断测试后，原来这是CMake的"变量污染"机制在捣乱。

<!--more-->

## 问题复现

首先我们新建一个C++的`hello world`项目，`CMakeList.txt`内容如下

``` c++
cmake_minimum_required(VERSION 3.0)

project(helloworld)

set(SRC main.cpp)

add_executable(hello_world ${SRC})

if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX ${PROJECT_SOURCE_DIR}/out)
endif()

install(TARGETS hello_world
        RUNTIME DESTINATION bin)
```

这是一个非常简单的CMake项目。我希望在用户没有指定安装目录时，`CMAKE_INSTALL_PREFIX`可以被自动设置为当前目录下的`out/`目录。然后我们创建一个`build`目录并进行编译安装。

``` bash
mkdir build && cd build && cmake .. && make && make install
```

执行之后我们可以发现，一切都很正常，在项目根目录生成了一个`out/bin`目录，里面有一个可执行文件`hello_world`。

然而，当我们在项目开发中，经常会需要对`CMakeList.txt`进行修改，比如对于这个测试项目，我希望修改生成的可执行文件名为`main`，这个时候我们将上面的`CMakeList.txt`中的所有`hello_world`修改为`main`即可。然后我们再次进入build目录。由于修改了`CMakeList.txt`，因此我们需要重新进行`cmake ..`操作然后编译和安装。需要执行的命令和上面一模一样。

然而这次我们发现，在`make install`时出错了，提示应该类似于下面这样

``` bash
Install the project...
-- Install configuration: ""
-- Installing: /usr/local/bin/main
CMake Error at cmake_install.cmake:47 (file):
  file INSTALL cannot copy file "/tmp/hello_world/build/main" to
  "/usr/local/bin/main".


make: *** [Makefile:118：install] 错误 1
```

明明在`cmake ..`时没有指定`-DCMAKE_INSTALL_PREFIX`，然而我们对`DCMAKE_INSTALL_PREFIX`的设置仿佛失效了，依然准备安装到`/usr/bin/`目录，导致安装出错。

## 问题分析

在经过多次测试，我找到了出错的规律，如果在每次首次执行`cmake`时，是不会出问题的，而当我执行了一次之后再次执行，安装路径必定会被还原为默认的系统路径，但如果将`cmake`生成的缓存文件清空，则又可以按照预期工作。这现象说明`cmake`生成的临时文件会对下一次`cmake`产生影响。

然而具体是如何影响的呢？这肯定和CMake的工作流程有关。经过搜索，我在[StackOverflow](https://stackoverflow.com/questions/39401003/why-there-are-two-buttons-in-gui-configure-and-generate-when-cli-does-all-in-one)上找到了答案。其中采纳答案的最后一段部分如下。

> CMake records information between runs in the variable cache. At the end of the run, it updates a file called CMakeCache.txt in the build directory. When you next run CMake, it reads in that cache to pre-populate various things so it doesn't have to recompute them (like finding libraries and other packages) and so that you don't have to supply custom options you want to override each time.

这一段大概意思是说，CMake会记录每次执行的变量信息到缓存文件`CMakeCache.txt`中。当用户再次执行CMake时，CMake将会在解析`CMakeList.txt`前先读取缓存文件中的变量信息加快处理速度。

由于读取的变量信息在解析`CMakeList.txt`之前，因此就相当于这些变量被预先设置了一个值，被缓存的值给"污染"了。因此在有缓存文件存在的情况下，其实`CMAKE_INSTALL_PREFIX`是有被设置的，而不是默认值。这就导致了非首次执行`cmake`，`CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT`为`false`。

然而又有一个问题来了。按照上述所说，首次执行的时候我们不是已经设置了`CMAKE_INSTALL_PREFIX`了么？我们设置的值应该会被缓存到缓存文件才对啊。为什么第二次执行读取到的缓存值还是默认系统安装目录呢？这说明我们对`CMAKE_INSTALL_PREFIX`的设置并没有写入到缓存文件。再次查阅CMake文档中关于`CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT`的内容，发现其给出的示例如下。

``` cmake
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX "/my/default" CACHE PATH "..." FORCE)
endif()
```

对比我们的测试项目中的写法，发现`set`后面多了几个参数。相信大家看到`CACHE`应该就猜到是怎么回事了。
再次查阅CMake文档中关于`set`的内容，在`Set Cache Entry`标题下有详细的说明。为了将设置信息写入缓存文件，我们需要使用如下形式的`set`方法。

``` cmake
set(<variable> <value>... CACHE <type> <docstring> [FORCE])
```

## 问题解决

再回到我们的测试项目上来，通过上述说明，我们已经知道问题的根源，解决办法就是将原来的`CMakeList.txt`中对`CMAKE_INSTALL_PREFIX`的设置一行修改为：

``` cmake
set(CMAKE_INSTALL_PREFIX ${PROJECT_SOURCE_DIR}/out CACHE PATH "default install prefix" FORCE)
```

重新`cmake`即可。

## 问题反思

1. 官方文档往往是解决问题的关键，一定要仔细阅读文档上的内容，包括示例，不放过任何一点细节。
2. 知其然更要知其所以然。由于网上和CMake相关的资料绝大部分都是讲述如何使用CMake而不是CMake会做些什么，尤其是中文网站。这导致我一直都不知道CMake会在执行时先去读取缓存文件这一机制。一知半解，最为致命。
