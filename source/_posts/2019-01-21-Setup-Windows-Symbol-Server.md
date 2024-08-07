---
title: 搭建Windows符号服务器
author: 张帆
tags:
  - 运维
date: 2019-01-21 17:03:04
---

在Windows桌面软件开发完成并发布给用户使用后，难免会遇到崩溃问题。通过`CrashDump`文件来分析崩溃问题是一个行之有效的途径。我们可以在程序崩溃时[在客户电脑上生成CrashDump文件](https://docs.microsoft.com/en-us/windows/desktop/wer/collecting-user-mode-dumps)并收集这些文件。但要打开并[分析CrashDump](https://docs.microsoft.com/en-us/windows/desktop/dxtecharts/crash-dump-analysis)文件，我们还需要产生`CrashDump`文件的可执行文件（后续称为`PE`文件，包括`exe`和`dll`文件）以及编译该`PE`文件时产生的符号文件（`PDB`）。比较基础的做法一般是在从客户电脑上拿到`CrashDump`文件和可执行文件后，再根据可执行文件的版本，编译时间等信息从构建服务器等地方找到相对应的`PDB`文件。这样带来的一个问题是我们需要手动做文件的匹配，由于`PE`文件，`PDB`文件和`CrashDump`文件三者是严格匹配的，即使是相同的代码，每次编译产生的文件都不一样，因此手动匹配稍有差错就会找不到正确的符号，导致无法分析`CrashDump`。为了解决这一问题，一个更好的方法是搭建一个符号服务器，在符号服务器上保存每次自动构建出的`PE`文件和`PDB`文件，让调试器根据`CrashDump`文件自动识别和匹配。下面将会介绍如何搭建一个私有的符号服务器。

<!--more-->

## 服务器环境配置

- Debugging Tools for Windows

创建、添加和删除符号需要用到Windows提供的工具`symstore`，该工具包含在[Debugging Tools for Windows](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/)包中，安装完成后需要将`symstore`程序所在目录（我的目录为`C:\Program Files (x86)\Windows Kits\10\Debuggers\x64`）添加到环境变量的`path`路径

- `IIS`

我们需要`IIS`来搭建一个提供文件访问的HTTP服务器用于符号的对外访问。安装`IIS`的方法网上已有很多教程，这里不再赘述。操作路径为控制面板->程序与功能->启用或关闭WIndows功能->勾选`Internet Information Service`。

## 存储符号

1. 创建存放符号的文件夹。由于符号文件会占用大量的磁盘空间，因此我们需要选择一块较大的磁盘，创建一个文件夹用于存放符号。这里假设我使用文件夹路径的是`D:\Symbols`。
2. 生成符号文件。我们在编译过程中，一般只会在编译Debug版本时会生成PDB文件，若需要[为Release版本生成PDB文件](https://www.wintellect.com/correctly-creating-native-c-release-build-pdbs)，需要在构建时添加一些参数。若使用qmake作为构建工具，则在pro文件中添加两行即可。

 ``` qmake
 QMAKE_LFLAGS_RELEASE += /debug /opt:ref
 QMAKE_CFLAGS_RELEASE += /Zi
 ```

3. 存储生成的`PE`和`PDB`文件。调试分析`CrashDump`文件需要`PE`和`PDB`文件，因此我们需要将生成的`PE`和`PDB`文件存储到符号库里。需要使用到的命令为`symstore`。利用该命令添加一个文件到符号库的命令如下

 ```
 symstore add /f FilePath /s StorePath /t Product [/v Version] [/c Comment]
 symstore add /r /f DirPath /s StorePath /t Product [/v Version] [/c Comment]
 ```

第一条命令用于添加某个特定的文件，第二个命令用于递归的搜索指定目录下的`PE`或`PDB`文件并存储。其中，`FilePath`为要存储的`PDB`或`PE`的路径，可以为相对路径或绝对路径。`DirPath`为包含`PE`或`PDB`文件的目录。`StorePath`为符号文件的保存路径，即我们刚创建的文件夹的路径，可以为本地路径，用于保存到本机的其他目录或网络路径，用于保存到其他服务器上。`Product`为产品名，一般为我们项目工程名，`Version`和`Commect`为可选项，分别为版本和描述信息。
执行该命令后我们会在符号文件存放目录看到通过一定规则存储的文件。我们可以将该命令添加到自动构建脚本中。


## 搭建HTTP服务

我们需要提供对外访问符号文件的途径，最简便的方式就是使用`IIS`创建一个`HTTP`服务并对外开放。使用搭`SII`建HTTP服务器的教程网上很多，这里简述以下。

1. 打开`IIS`，选择网站->右键“添加网站”
2. 输入网站名称，如`Symbols`，选择刚创建的符号文件存储目录为物理路径，其他保持默认，点击确定。这里需要注意的一点是若使用默认的80端口，需要将`Default Web Site`停止或删除，因为80端口已被其占用。
3. 在网站主页中打开`MIME类型`，右侧添加，文件扩展名输入`PDB`,MIME类型输入`application/octet-stream`。这一步是为了允许从该站点访问读取`PDB`类型的文件。
4. 设置权限。我们创建的网站默认情况下是不允许匿名用户访问的，为了允许调试器下载符号，我们需要添加一定的权限。点击右键我们的网站->编辑权限->选择`安全`选项卡->添加`IIS_IUSRS`用户或`everyone`用户并赋予`Read & execute`，`List folder contents`和`Read`三项权限。
5. 左侧右键刚创建的网站->管理网站->重新启动。

## 在Visual Studio中使用

我们将崩溃产生的`CrashDump`文件使用`Visual Studio`打开，然后选择设置符号路径，将我们的HTTP网站的网址（一般为`http://服务器IP`）填进去即可调试


## 过期符号的清理

由于每次编译均会生成符号文件，如果不定期清理，会大量占用磁盘空间。`symstore`没有清理过期文件的功能，但每次成功调用`symstore`命令后都会有一个唯一的操作ID，我们可以通过该ID来删除添加的文件。在符号文件存储目录下的`000Admin`目录里，有一个`history.txt`文件，该文件记录了所有`symstore`命令的历史记录，我们可以通过分析该文件来获取操作ID，添加时间等信息，确定需要清理的文件。`symstore`删除指定操作添加的文件的命令如下

 ``` cmd
 `symstore` del /i ID /s StorePath
 ```

这样是非常繁琐的，需要手动操作，但目前并没有找到一个很好的方式去批量或自动处理这一问题。可能需要实现一个程序去分析`history.txt`并执行相关处理。
