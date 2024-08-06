---
title: 修改Windows系统环境变量
author: 张帆
tags:
  - Windows
abbrlink: 22206
date: 2019-04-27 09:58:16
---

每个接触过Windows系统的开发人员都会接触到环境变量，相当一部分软件安装完成后必做的一步操作就是设置环境变量。Windows下的环境变量分为用户环境变量和系统环境变量。本文主要分享如何正确设置系统环境变量。

## GUI界面

这是我们最常用的方式，即右键此电脑->属性->高级系统设置->环境变量，然后在系统变量栏中修改系统环境变量。该方法十分简单，但我们有时候可能需要通过程序来修改，又该怎么做呢？

![环境变量](https://blog-1251989759.cos.ap-guangzhou.myqcloud.com/2019-04-27/modify_system_environment_variable.jpg)


## 命令行

在`CMD`中，我们可以通过`set`命令来设置环境变量，`powershell`中也可以通过`$ENV:key=value`的形式设置环境变量，但这只能影响当前命令行环境，命令行程序退出后就失效了。我们可以通过`SetX`来修改系统环境变量。官方描述如下：

> 在用户或系统环境创建或修改环境变量。能基于参数、注册表项或文件输入设置变量。

默认情况下，该命令是修改当前用户的的环境变量，如果我们需要修改系统环境变量，可以使用`/M`参数。但正如官方描述所说，该命令只能 *创建或修改* 环境变量，也就是说该命令无法用于删除一个环境变量。

## 注册表

根据微软的[官方文档](https://docs.microsoft.com/zh-cn/windows/desktop/ProcThread/environment-variables)， Windows系统环境变量其实是保存在注册表中的，路径为`HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Session Manager\Environment`，我们可以手动或通过代码修改注册表的方式来修改系统环境变量。

> To programmatically add or modify system environment variables, add them to the HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Session Manager\Environment registry key, then broadcast a WM_SETTINGCHANGE message with lParam set to the string "Environment". This allows applications, such as the shell, to pick up your updates.

修改完成后我们需要广播一个消息类型为`WM_SETTINGCHANGE`，`lParam`参数为`"Environment"`的窗口消息，该消息将会通知应用程序获取我们的更新。经过测试我发现，目前Windows下的`explorer.exe`是会处理这个消息然后更新其环境变量。这也就解释了之前一直以来的一个困惑，就是在修改完系统环境变量后，新启动的程序有的可以获取到更新后的环境变量信息，而有的必须则不行。这其实是因为新进程是会继承父进程的环境变量，如果父进程处理了上述的窗口消息且主动更新了，那么子进程就会使用更新后的环境变量信息，例如通过`explorer.exe`(这是Windows的窗口管理器，包括开始菜单，桌面等都是由该程序管理)启动的程序。而如果某些程序没有处理这个窗口消息，那么其不会更新自己的环境变量，进而导致通过其启动的新进程也就是旧的环境变量信息。例如我使用的一款启动器软件就没有处理这个消息，导致如果我通过其启动的软件都是旧环境变量信息，只能重新启动该启动器或重启系统。

在发送这条窗口消息时，还有两个需要注意的问题。

1. 如何将一个字符串设置给`lParam`参数？

 Windows下发送窗口消息的几种方法中，参数lParam都是是`LPARAM`类型，在Win10 x64系统下，其定义如下

 ``` cpp
 typedef LONG_PTR            LPARAM;
 typedef _W64 long LONG_PTR, *PLONG_PTR;
 ```

 也就是说lParam是一个整型，那如何将一个字符串赋值给整型呢？其实通过中间的`LONG_PTR`我们可以看出，这里Windows是用一个整型来存储字符串的地址。我们可以直接将字符串强转成一个整型。

 ``` cpp
 SendMessage(HWND_BROADCAST, WM_SETTINGCHANGE, NULL, LPARAM("Environment"));
 ```

2. 调用无效？

 上面的代码调用看上去已经满足文档上的要求了，但在实际测试时，在`Win10 中文版`系统上，发送该消息后，再通过`explorer.exe`启动新程序，新的程序还是旧的环境变量信息。难道文档错了么？这个问题困扰了我非常久，后来才发现这是由于Windows为了兼容不同的字符编码环境，绝大部分API都提供了三套方法，除了普通的形式外，还有一个以`A`结尾，一个以`W`的方法，其中`A`代表`ASCII`码，即窄字符串版本，`W`代表宽字符版本。普通形式的方法实际上会根据是否有定义`__UNICODE__`这个宏来`define`为以`A`结尾或以`W`结尾的方法。如`SendMessage`的定义如下：

 ```
 #ifdef UNICODE
 #define SendMessage  SendMessageW
 #else
 #define SendMessage  SendMessageA
 #endif // !UNICODE
 ```

 在我的电脑环境下是有定义`UNICODE`宏的，因此`SendMessage`方法实际上是调用了`SendMessageW`方法，然而第四个参数`"Environment"`是一个`ASCII`字符串，这就导致参数和方法不匹配，进而导致该消息没有被正确处理。下面是几种可行的修改方式

 ```
 SendMessageA(HWND_BROADCAST, WM_SETTINGCHANGE, NULL, LPARAM("Environment")); // 明确指定使用
 SendMessageW(HWND_BROADCAST, WM_SETTINGCHANGE, NULL, LPARAM(L"Environment")); // 明确指定使用宽字符版本
 SendMessage(HWND_BROADCAST, WM_SETTINGCHANGE, NULL, LPARAM(_T"Environment")); // _T是一个根据UNICODE宏调整的宏，分别会被定义为空或L
 ```

## 总结

下面是修改系统环境变量的几种方法的对比

| 方式                     | 支持程序调用 | 支持手动修改 | 添加 | 修改 | 删除 | 即时生效 |
| ---                      | ---          | ---          | ---  | ---  | --   | ---      |
| GUI界面                  |              | √            | √    | √    | √    | √        |
| 命令行                   | √            | √            | √    | √    |      | √        |
| 修改注册表并发送窗体消息 | √            |              | √    | √    | √    | √        |

<script src="https://utteranc.es/client.js"
        repo="xyz1001/xyz1001.github.io"
        issue-term="title"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
