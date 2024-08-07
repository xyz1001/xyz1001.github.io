---
title: 踩坑记之特殊的头文件名
author: 张帆
tags:
  - 踩坑记
  - MSVC
abbrlink: 30205
date: 2020-08-29 15:25:09
---

## 问题现象

windows 下编译[ Share 项目](https://github.com/xyz1001/BlogExamples/tree/master/Share)时，编译器报各种莫名错误

<!--more-->

```
[0/1 ?/sec] Re-running CMake...
-- Configuring done
-- Generating done
-- Build files have been written to: D:/Code/build-Share-Desktop_Qt_5_12_5_MSVC2017_32bit-Debug
[1/6 8.9/sec] Automatic MOC and UIC for target Share
[2/6 1.5/sec] Building CXX object CMakeFiles\Share.dir\Share_autogen\mocs_compilation.cpp.obj
FAILED: CMakeFiles/Share.dir/Share_autogen/mocs_compilation.cpp.obj 
C:\PROGRA~2\MICROS~1\2017\COMMUN~1\VC\Tools\MSVC\1416~1.270\bin\HostX86\x86\cl.exe  /nologo /TP -DQT_CORE_LIB -DQT_GUI_LIB -DQT_WIDGETS_LIB -I. -ID:\Code\Share -IShare_autogen\include -IC:\Qt\5.12.5\msvc2017\include -IC:\Qt\5.12.5\msvc2017\include\QtWidgets -IC:\Qt\5.12.5\msvc2017\include\QtGui -IC:\Qt\5.12.5\msvc2017\include\QtANGLE -IC:\Qt\5.12.5\msvc2017\include\QtCore -IC:\Qt\5.12.5\msvc2017\.\mkspecs\win32-msvc /DWIN32 /D_WINDOWS /W3 /GR /EHsc /showIncludes /MDd /Zi /Ob0 /Od /RTC1 /showIncludes /FoCMakeFiles\Share.dir\Share_autogen\mocs_compilation.cpp.obj /FdCMakeFiles\Share.dir\ /FS -c Share_autogen\mocs_compilation.cpp
D:\Code\Share\share.h(6): error C2504: “QObject”: 未定义基类
D:\Code\Share\share.h(7): error C2027: 使用了未定义类型“QString”
C:\Qt\5.12.5\msvc2017\include\QtCore/qchar.h(48): note: 参见“QString”的声明
d:\code\build-share-desktop_qt_5_12_5_msvc2017_32bit-debug\share_autogen\EWIEGA46WW/moc_share.cpp(85): error C2352: “QObject::qt_metacast”: 非静态成员函数的非法调用
c:\qt\5.12.5\msvc2017\include\qtcore\qobject.h(119): note: 参见“QObject::qt_metacast”的声明
d:\code\build-share-desktop_qt_5_12_5_msvc2017_32bit-debug\share_autogen\EWIEGA46WW/moc_share.cpp(90): error C2352: “QObject::qt_metacall”: 非静态成员函数的非法调用
c:\qt\5.12.5\msvc2017\include\qtcore\qobject.h(119): note: 参见“QObject::qt_metacall”的声明
[3/6 2.2/sec] Building CXX object CMakeFiles\Share.dir\mainwindow.cpp.obj
FAILED: CMakeFiles/Share.dir/mainwindow.cpp.obj 
C:\PROGRA~2\MICROS~1\2017\COMMUN~1\VC\Tools\MSVC\1416~1.270\bin\HostX86\x86\cl.exe  /nologo /TP -DQT_CORE_LIB -DQT_GUI_LIB -DQT_WIDGETS_LIB -I. -ID:\Code\Share -IShare_autogen\include -IC:\Qt\5.12.5\msvc2017\include -IC:\Qt\5.12.5\msvc2017\include\QtWidgets -IC:\Qt\5.12.5\msvc2017\include\QtGui -IC:\Qt\5.12.5\msvc2017\include\QtANGLE -IC:\Qt\5.12.5\msvc2017\include\QtCore -IC:\Qt\5.12.5\msvc2017\.\mkspecs\win32-msvc /DWIN32 /D_WINDOWS /W3 /GR /EHsc /showIncludes /MDd /Zi /Ob0 /Od /RTC1 /showIncludes /FoCMakeFiles\Share.dir\mainwindow.cpp.obj /FdCMakeFiles\Share.dir\ /FS -c D:\Code\Share\mainwindow.cpp
D:\Code\Share\share.h(6): error C2504: “QObject”: 未定义基类
D:\Code\Share\share.h(7): error C2027: 使用了未定义类型“QString”
C:\Qt\5.12.5\msvc2017\include\QtCore/qchar.h(48): note: 参见“QString”的声明
[4/6 3.0/sec] Building CXX object CMakeFiles\Share.dir\main.cpp.obj
FAILED: CMakeFiles/Share.dir/main.cpp.obj 
C:\PROGRA~2\MICROS~1\2017\COMMUN~1\VC\Tools\MSVC\1416~1.270\bin\HostX86\x86\cl.exe  /nologo /TP -DQT_CORE_LIB -DQT_GUI_LIB -DQT_WIDGETS_LIB -I. -ID:\Code\Share -IShare_autogen\include -IC:\Qt\5.12.5\msvc2017\include -IC:\Qt\5.12.5\msvc2017\include\QtWidgets -IC:\Qt\5.12.5\msvc2017\include\QtGui -IC:\Qt\5.12.5\msvc2017\include\QtANGLE -IC:\Qt\5.12.5\msvc2017\include\QtCore -IC:\Qt\5.12.5\msvc2017\.\mkspecs\win32-msvc /DWIN32 /D_WINDOWS /W3 /GR /EHsc /showIncludes /MDd /Zi /Ob0 /Od /RTC1 /showIncludes /FoCMakeFiles\Share.dir\main.cpp.obj /FdCMakeFiles\Share.dir\ /FS -c D:\Code\Share\main.cpp
D:\Code\Share\share.h(6): error C2504: “QObject”: 未定义基类
D:\Code\Share\share.h(7): error C2027: 使用了未定义类型“QString”
C:\Qt\5.12.5\msvc2017\include\QtCore/qchar.h(48): note: 参见“QString”的声明
[5/6 3.5/sec] Building CXX object CMakeFiles\Share.dir\share.cpp.obj
ninja: build stopped: subcommand failed.
15:08:49: 进程"C:\Users\87591\scoop\shims\cmake.exe"退出，退出代码 1 。
Error while building/deploying project Share (kit: Desktop Qt 5.12.5 MSVC2017 32bit)
When executing step "CMake Build"
15:08:49: Elapsed time: 00:02.
```

## 原因分析

首先新建一个Qt初始项目编译正常，排除开发环境问题。
接下来没有任何思路，只好逐步精简项目，发现移除`share.h`和`share.cpp`并移除其他文件中对`share.h`的包含后依然会报`share.h`中的错误，尝试修改`share.h`的名称找出哪个文件还依赖了该文件，发现改名后编译居然通过了。遂猜测该文件名可能有冲突。使用 everything 搜索`share.h`，发现同名文件`"C:\Program Files (x86)\Windows Kits\10\Include\10.0.16299.0\ucrt\share.h"`，位于标准库搜索路径中，表明该文件属于标准库头文件。
至此，问题原因找到，我们的`share.h`文件覆盖了标准库中的`share.h`，导致原本应该包含标准库中`share.h`变成包含我们提供的`share.h`。

## 解决方法

修改share.h的文件名为其他名称后解决。

但事情到这里还没彻底弄清楚，按照C++头文件包含规则，如果是双引号则优先在当前项目路径下查找头文件，如果是尖括号则优先从系统目录查找。这里已经可以确认示例项目中没有包含`share.h`文件，也就是说是程序依赖的库间接依赖了该头文件。而对于库来说，其本来目的肯定是要包含系统目录中的`share.h`，那么理论上编程习惯没有问题的话，应使用`#inlcude <share.h>`，这样会优先查找到系统库中的`share.h`，不会出问题。而根据目前现象来看，有两种猜测，要么是某个库中对`share.h`的依赖写成了`#include "share.h"`，要么是 MSVC 编译器对`""`和`<>`的处理存在问题。所以我们要找到到底是哪个文件包含了`share.h`。
MSVC 提供了`/showIncludes`编译选项，用于打印头文件包含关系，我们用 Visual Studio 打开（用 Qt Creator 测试并没有打印），发现大致包含关系如下：
```
MainWindow.h > QMainWindow > ... > qbytearray.h > string > ... > xiosbase > share.h
 ```
 所以直接包含`share.h`的是`xiosbase`这个文件，打开后发现其是通过`#include <share.h>`来包含的，所以猜测一不对，因此只可能是 MSVC 编译器对`""`和`<>`的处理是存在问题的，但通过附件中的[ TestInclude ](https://github.com/xyz1001/BlogExamples/tree/master/TestInclude)项目又正常，猜测可能是在某些特定条件下才会触发该 bug，但到这里已经无法再继续追下去了，只能作罢。
