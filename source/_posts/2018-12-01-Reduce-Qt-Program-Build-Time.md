---
title: 加快Qt程序的编译速度
author: 张帆
tags:
  - Qt
date: 2018-12-01 09:02:39
---

在项目开发中，随着产品的更新迭代，项目会变得越来越大。项目的增大就会导致编译速度也会变得越来越慢。本人负责的一个项目的代码量如下

```
github.com/AlDanial/cloc v 1.80  T=0.07 s (2139.2 files/s, 210543.4 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
C++                             73           1560            453           7993
C/C++ Header                    78           1167            865           2919
Markdown                         1              1              0              2
-------------------------------------------------------------------------------
SUM:                           152           2728           1318          10914
-------------------------------------------------------------------------------
```

在不同的环境下全新构建一次耗费时间（单位：秒）如下

|        | Qt Creator | Visual Studio 2017 | 命令行 |
| ---    | ---        | ---                | ---    |
| 本地   | 81         | 241                | 246    |
| 服务器 | -          | -                  | 183    |

<!--more-->

其中编译环境如下：

|      | 本地             | 服务器              |
| ---  | ---              | ---                 |
| 系统 | Windows 10 1803  | Windows Server 2016 |
| CPU  | i5-7300HQ 2.5GHz | i7-4790 3.6GHz      |
| 硬盘 | 固态硬盘         | 机械硬盘            |
| 内存 | 8G               | 16G                 |

构建命令行如下

``` bat
qmake ..
nmake release
```

可以看到，在`Qt Creator`中构建速度相对快很多，但由于`Qt Creator`目前不支持远程调试等重要功能，因此经常需要使用`Visual Studio 2017`进行编译。但是每次都过qmake生成或更新VS工程都会导致`Visual Studio 2017`将整个项目重新构建一遍，而本地`Visual Studio 2017`全新构建一次需要4分多钟。在服务器上，由于该项目需要同时生成两个版本的安装包，整个流程下来需要10分多钟。最近刚好看到了[姚冬](https://www.zhihu.com/people/yao-dong-27)大神的关于Qt编译速度为什么这么慢的[回答](https://www.zhihu.com/question/23045749/answer/23659031)，因此整理了下加快Qt程序（使用qmake作为构建工具）编译速度的三个措施。

## 优化措施

### 使用预编译头

预编译头在VC项目很常见，通过VS默认创建的C++项目就是带有预编译头的。实际上除了`MSVC`，目前主流的编译器，如gcc，clang均已支持了预编译头。关于预编译头的介绍可以参考维基百科[预编译头](https://zh.wikipedia.org/wiki/%E9%A2%84%E7%BC%96%E8%AF%91%E5%A4%B4)。简单来说就是预编译头可以将头文件提前进行编译，这样就不用在每个编译单元中重复编译了。Qt，准确来说是qmake提供了非常简单的可跨平台的方式启用预编译头，[官方文档](http://doc.qt.io/qt-5/qmake-precompiledheaders.html)中有非常详细的使用步骤。下面是对官方文档的关键信息的翻译。

#### 支持的构建工具

qmake支持在以下平台和构建环境中预编译头

- Windows
  - nmake
  - Visual Studio projects (VS 2008 and later)
- macOS, iOS, tvOS, and watchOS
  - Makefile
  - Xcode
- Unix
  - GCC 3.4 and above

可以看到，qmake对预编译头的支持已经覆盖了绝大部分常用的场景，跨平台性非常好。

#### 启用步骤

1. 创建预编译头文件

 新建一个头文件，如`precompile.h`，然后将项目中所有用到的稳定和静态的头文件添加到该文件中，内容类似于

 ``` cpp

 extern "C" {
 // 在此添加C头文件
 include "libavcodec/avcodec.h"
 ...
 }

 // 在此添加C++头文件
 #include <cstdlib>   //C++标准库头文件
 #include <iostream>
 #include <vector>
 ...

 #include <QApplication> // Qt库头文件
 #include <QPushButton>
 #include <QLabel>
 ...

 #include "thirdparty/include/libmain.h" // 第三方库头文件
 ...

 #include "my_stable_class.h" // 自己项目中稳定的头文件
 ...
 ```

 在Qt官方文档中，对于哪些头文件适合放入预编译头中提到了两个关键词，一个是`stable`，一个是`static`，个人的理解如下

 - `stable`：该头文件很少改动，如C++标准库，Qt库等，`static`表示该头文件
 - `static`：该头文件每次编译生成的代码是固定的，而不会受到诸如环境变量、宏定义的影响而经常变动

 另外，由于预编译头是在该头文件会被多个源文件包含时才会有节省编译时间的效果，因此对于只会被单一源文件使用的头文件，也没有很大必要放入预编译头。

 这里有一个小技巧可以快速提取出项目中使用到的头文件，即我们可以通过搜索工具搜索`include`关键词，然后将搜索结果进行排序，去重，筛选。

2. 添加构建选项

 在`.pro`文件中，添加以下两行代码

 ```
 CONFIG += precompile_header
 PRECOMPILED_HEADER = precompile.h
 ```

这样我们就启用了预编译头。可以看到，在使用qmake的项目中开启预编译头十分方便。而且是非入侵式，即我们不需要对已有的代码做任何修改。

补充：cmake中有一个第三方库可以比较方便的提供跨平台预编译头的支持，参见[cotire](https://github.com/sakra/cotire)，不过我在Linux试了下效果并不好，可能是工程比较小

### 并行编译

开启并行编译（多线程编译）是加快编译速度最直接有效的方式。在不同的平台和环境下开启并行编译的方式不尽相同。

#### MSVC

在`.pro`文件中添加以下代码

```
*msvc* {
    QMAKE_CXXFLAGS += /MP
}
```

#### g++/clang

- 使用`Qt Creator`

 左侧`项目` - `Build` - `构建步骤` -> `Make` -> `Make参数` -> `添加-j?(?表示你想使用的线程数)`，此处的线程数推荐和CPU的线程数相匹配。
 ![开启并行编译](enable_mp.png)

- 使用命令行

 直接在`make`命令后添加`-j?(?表示你想使用的线程数)`

### 使用jom代替nmake

在Windows下，默认的make工具是微软提供的nmake，但是nmake缺少对独立命令的并行执行的支持。这里的并行执行和上面的并行编译看上去很像，实际上是完全不同的。

- 并行编译：指在编译阶段，使用多线程去进行源代码的编译，即同时编译多个C++编译单元
- 并行执行：由于Qt项目在构建过程中并不是仅仅执行C++代码编译命令（在MSVC下即`cl`命令），还会调用如`moc`（元对象编译器），`uic`（UI文件编译器），`rcc`（资源文件编译器）等多个工具对源代码进行预处理，这些命令都是互相独立的，完全可以并行执行，从而打到节省构建时间的目的。

在上面的构建时间对比中我们可以很明显的看到，使用 `Qt Creator`构建比其他两种方式要快得多，这是因为`Qt Creator`默认使用了`jom`来代替了`nmake`。[jom](https://wiki.qt.io/Jom)是对nmake的克隆，添加了对独立命令的并行执行功能。其用法和`nmake`几乎完全一样，因此我们仅需要将`nmake`命令替换为`jom`即可。然而很不幸的是，`Visual Studio`并没有提供可以替换nmake的途径，因此我们如果使用`Visual Studio`，是无法享受`jom`带来的时间节省的。
jom会根据我们电脑当前CPU空闲的线程数动态确定构建时所使用的线程数，当然，我们也可以通过`-j ?`(?表示你想使用的线程数)来手动指定使用的线程数。

## 优化效果

以下是在本地不同环境下开启不同优化处理后的编译时间对比

|          | Qt Creator * | Visual Studio 2017 | 命令行 |
| ---      | ---          | ---                | ---    |
| 默认     | 81           | 241                | 246    |
| 预编译头 | 39           | 87                 | 94     |
| 并行编译 | 81           | 105                | 130    |
| 使用Jom  | -            | -                  | 77     |
| 同时开启 | 40           | 66                 | 40     |

*: `Qt Creator`默认使用Jom

分析上表我们可以得出以下结论

- `Qt Creator`默认使用了Jom代替nmake
- Jom已经默认提供了并行编译的支持，因此并行编译开启与否作用不大
- `Visual Studio 2017`除了不能使用Jom，其他情况下速度要比相同条件下的命令行快，可能是其编译参数有差异或者存在其它的优化措施，这个后续可以再深入研究

## 参考

- [Jom](https://wiki.qt.io/Jom)
- [预编译头](https://zh.wikipedia.org/zh-hans/%E9%A2%84%E7%BC%96%E8%AF%91%E5%A4%B4)
- [为什么 Qt Creator 的编译如此之慢？ - 姚冬的回答 - 知乎](https://www.zhihu.com/question/23045749/answer/23659031)
- [How to Make Your C++ Qt Project Build 10x Faster with 4 Optimizations](https://developex.com/blog/how-to-make-your-c-qt-project-build-10x-faster-with-4-optimizations/)
- [Speeding up Visual C++ Qt Builds](http://blog.qt.io/blog/2009/03/27/speeding-up-visual-c-qt-builds/)
