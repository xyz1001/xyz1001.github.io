---
title: Linux软件调试体验优化
author: 张帆
tags:
  - Linux
  - 调试
date: 2024-08-19 21:29:33
---


## 背景

我们内部的应用软件发布后，在用户设备上经常会出现崩溃等问题导致闪退，对于这些用户现场发生的问题只能通过日志进行分析，难以排查出问题。日志可能一方面不一定包含异常情况相关的信息，另一方面由于文件缓存等原因，导致崩溃前一段时间的日志不一定及时写入文件，信息存在丢失，因此往往难以排查出问题。

<!--more-->

## 方案

对于崩溃类问题，崩溃点的转储dump信息是极其重要且有效的问题分析突破点。我们可以从dump文件中读取到包括系统环境，进程环境，所有线程（包括崩溃线程）的函数堆栈和代码行号，变量数据等在内的众多信息，比分析日志要精准很多。
因此，我们需要搭建一套从符号管理，dump收集到崩溃分析的全流程工具链，最终达到以下效果：

1. 代码开发完成，上传后触发自动构建
2. 自动购建时生成包含完整符号信息的二进制文件
3. 生成的文件自动上传至符号服务器进行统一保存
4. 应用发布后，用户现场出现崩溃时自动生成对应的dump文件
5. dump文件生成后自动上传至dump收集服务器
6. dump收集服务器自动整理分析dump并分派任务给到对应负责人
7. 对应负责人分析并处理问题，更新代码，返回第一步

## 实施

### 生成符号

当前项目项目发布版本构建使用的是CMake提供的Release默认参数，不会生成调试符号。

> (gdb) bt
> #0  __GI___libc_read (nbytes=1024, buf=0x5555555fcdf0, fd=0) at ../sysdeps/unix/sysv/linux/read.c:26
> #1  __GI___libc_read (fd=0, buf=0x5555555fcdf0, nbytes=1024) at ../sysdeps/unix/sysv/linux/read.c:24
> #2  0x00007ffff508cc36 in _IO_new_file_underflow (fp=0x7ffff521aaa0 <_IO_2_1_stdin_>) at ./libio/libioP.h:947
> #3  0x00007ffff508dd96 in __GI__IO_default_uflow (fp=0x7ffff521aaa0 <_IO_2_1_stdin_>) at ./libio/libioP.h:947
> #4  0x00007ffff5087a98 in _IO_getc (fp=0x7ffff521aaa0 <_IO_2_1_stdin_>) at ./libio/getc.c:40
> #5  0x00007ffff55128a1 in __gnu_cxx::stdio_sync_filebuf<char, std::char_traits<char> >::uflow() () from /lib/x86_64-linux-gnu/libstdc++.so.6
> #6  0x00007ffff5520f4e in std::istream::get() () from /lib/x86_64-linux-gnu/libstdc++.so.6
> #7  0x0000555555569f23 in test1() ()
> #8  0x00005555555685ed in main ()
> (gdb) frame 7
> #7  0x0000555555569f23 in test1() ()
> (gdb) i locals
> No symbol table info available.

可以看到，在我们内部模块的调用栈（栈7和栈8）没有具体的符号信息（函数签名和代码行号），也没有对应栈帧的局部变量。这里需要注意的是，这里之所以能看到函数名，仅仅只是因为我们错误的把所有的符号都进行了导出，对应内联函数等，这里显示的符号是可能出错的，同时导出所有符号也导致链接时间特别长。另外这类符号一旦strip就会无法查看了。
因此，为了提高dump文件的可分析性，我们需要在构建release版本时，也和debug版本一样，生成对应的调试符号。这里可能存在的一个误区是只有debug版本才能有调试符号，或者release版本生成了调试符号会影响性能，实际上是否进行优化（由-Ox编译参数进行控制）和是否生成调试信息（由-g进行控制）二者是几乎正交的（高优化等级可能会导致部分变量和符号和实际代码不一致，影响调试）。因此开启调试符号生成是完全不影响代码优化的。

CMake内置了[RelWithDebInfo](https://cmake.org/cmake/help/latest/variable/CMAKE_BUILD_TYPE.html)这一构建类型，用于生成带调试符号的优化版本，我们可以通过CMAKE_BUILD_TYPE= RelWithDebInfo来指定。同样的，conan也对此做了支持，我们可以指定conan 配置中的build_type为RelWithDebInfo来实现。对应构建命令如下：

```
conan build . -pr:b default -pr:h default -s:h build_type=RelWithDebInfo
```

对应版本调试时即可看到调试符号。

> (gdb) bt
> #0  __GI___libc_read (nbytes=1024, buf=0x5555555fddf0, fd=0) at ../sysdeps/unix/sysv/linux/read.c:26
> #1  __GI___libc_read (fd=0, buf=0x5555555fddf0, nbytes=1024) at ../sysdeps/unix/sysv/linux/read.c:24
> #2  0x00007ffff508cc36 in _IO_new_file_underflow (fp=0x7ffff521aaa0 <_IO_2_1_stdin_>) at ./libio/libioP.h:947
> #3  0x00007ffff508dd96 in __GI__IO_default_uflow (fp=0x7ffff521aaa0 <_IO_2_1_stdin_>) at ./libio/libioP.h:947
> #4  0x00007ffff5087a98 in _IO_getc (fp=0x7ffff521aaa0 <_IO_2_1_stdin_>) at ./libio/getc.c:40
> #5  0x00007ffff55128a1 in __gnu_cxx::stdio_sync_filebuf<char, std::char_traits<char> >::uflow() () from /lib/x86_64-linux-gnu/libstdc++.so.6
> #6  0x00007ffff5520f4e in std::istream::get() () from /lib/x86_64-linux-gnu/libstdc++.so.6
> #7  0x000055555556cb82 in test1 () at /home/xxx/code/Demo/main.cpp:12
> #8  0x000055555556b04d in main () at /home/xxx/code/Demo/main.cpp:59
> (gdb) frame 7
> #7  0x000055555556cb82 in test1 () at /home/xxx/code/Demo/main.cpp:12
> 12              std::cin.get();
> (gdb) i locals
> ec = std::error_code = {std::_V2::error_category: 0}
> project_path = filesystem::path "/home/xxx/code/Demo" = {[root-directory] = "/", [1] = "home", [2] = "xxx", [3] = "code", [4] = "Demo"}

但这样会导致二进制文件体积增大，同时这些详细的调试符号也是不能对外公开的，因此我们需要在发布给用户的软件打包时对这些文件进行strip操作

> $ ls -alh Demo
> -rwxrwxr-x 1 xxx xxx 9.3M  8月  8 11:53 Demo
> $ file Demo
> Demo: ELF 64-bit LSB pie executable, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c98fed03ec124428f25bdf7b2e6fd047047b7edf, for GNU/Linux 3.2.0, with debug_info, not stripped
> $ strip Demo
> $ ls -alh Demo
> -rwxrwxr-x 1 xxx xxx 327K  8月  8 15:25 Demo
> $ file Demo
> Demo: ELF 64-bit LSB pie executable, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c98fed03ec124428f25bdf7b2e6fd047047b7edf, for GNU/Linux 3.2.0, stripped

那如何保存未strip前的文件用于调试呢，这就需要用到符号服务器。

### 搭建符号服务器

[debuginfod](https://sourceware.org/elfutils/Debuginfod.html)是一个基于HTTP协议的用于管理调试符号的服务端工具，包括gdb，QtCreator在内的绝大部分Linux调试工具都提供了对该工具的支持，可以在调试时向debuginfod获取地址对应的符号信息。debugfinfod功能类似于[Windows的符号服务器](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/using-a-symbol-server)，但其推出有点过于晚了。
debugfinfod的部署比较简单，只要在服务器用于保存符号文件的目录下，执行以下命令

```bash
debuginfod -F .
```

debuginfod会自动扫描该目录下的文件。运行后，debuginfod会开启一个监听在8002端口的HTTP服务。
在调试时，我们需要先设置环境变量DEBUGINFOD_URLS为服务器上debuginfod的HTTP链接。

```bash
export DEBUGINFOD_URLS=http://<server IP>:8002/
```

然后启动调试软件，如gdb，其会自动识别该环境变量。gdb在检测到该环境变量时会提示开启debuginfod

> This GDB supports auto-downloading debuginfo from the following URLs
> http://localhost:8002
> Enable debuginfod for this session? (y or [n]) y
> Debuginfod has been enabled.
> To make this setting permanent, add 'set debuginfod enabled on' to .gdbinit.
> Downloading 9.21 MB separate debug info for /home/xxx/code/Demo/build/RelWithDebInfo/examples/Demo/Demo
> Reading symbols from /home/xxx/.cache/debuginfod_client/c98fed03ec124428f25bdf7b2e6fd047047b7edf/debuginfo...

我们需要使用一台磁盘空间较大的服务器，在上面选择一个目录用于保存符号文件，同时运行debuginfod命令即可。那符号文件从哪里来，肯定不能编译一次复制一次，我们需要在自动构建时自动上传。

### 符号自动上传

对外发布的软件是通过自动构建服务器编译出来的，因此我们可以在自动构时将编译出来的动态库和可执行程序上传到符号服务器。若构建服务器和符号服务器是一台硬件设备，那我们直接通过cp命令复制过去即可，否则就需要在符号服务器上部署文件上传服务（例如基于HTTP的文件上传服务），我们可以直接我们可以直接使用网上现成的脚本，例如[SimpleHTTPServer](https://stackoverflow.com/a/58255859/6930691)。
那如何在编译完成后自动触发上传呢？一种方式是根据CI工具，修改自动构建脚本。考虑到我们目前所有项目都使用conan进行构建，因此更加方便的方式是通过编写conan hook脚本来实现。conan提供了对打包整个过程的各个步骤的hook支持。我们只要在打包完成时执行脚本进行上传即可。具体代码如下

```python
import os
import platform
import shutil
import hashlib

def post_package(conanfile):
    if platform.system() != "Linux":
        return

    conanfile.output.info(f"post_package: {conanfile.name}/{conanfile.version}@{conanfile.user}/{conanfile.channel}")
    # develop默认不上传符号，避免过多的符号文件浪费符号服务器磁盘空间
    if conanfile.channel == "develop":
        conanfile.output.info("package is develop, skip upload symbol files")
        return

    # 非动态库，不上传符号
    if not conanfile.options.get_safe("shared"):
        conanfile.output.info("package is not shared, skip upload symbol files")
        return

    files = []
    for dirName, _, fileList in os.walk(conanfile.package_folder):
        for file in fileList:
            if ".so." in file:
                files.append(os.path.join(dirName, file))
                conanfile.output.info("Found so: %s" % os.path.join(dirName, file))
    if not files:
        conanfile.output.info("package has no so files, skip upload symbol files")
        return

    # 这里以构建服务器和符号服务器在同一台机器上为例，保存路径为/symbols
    # 由于存在同名文件，因此先计算文件md5，然后将文件复制到以md5命名的文件夹
    for file in files:
        md5 = hashlib.md5(open(file, 'rb').read()).hexdigest()
        os.makedirs(f"/symbols/{md5}", exist_ok=True)
        shutil.copy(file, f"/symbols/{md5}")
        conanfile.output.info(f"copy {file} to /symbols/{md5}")
```

若是构建服务器和符号服务器非同一台硬件设备，则修改最后四行为HTTP上传逻辑即可，需要注意的是HTTP server需要根据文件哈希创建不同的文件夹避免文件被覆盖。
然后我们将该文件命名为以`hook_`开头，`.py`为扩展名后，放到构建服务器的`~/.conan2/extensions/hooks`目录即可，conan会自动识别并启用。

### 软件打包

当我们在进行对外发布软件打包时，由于我们生成了调试符号，因此我们需要对发布的软件中的二进制文件进行strip操作。
这里将以下命令添加至应用的构建脚本，在安装目录执行即可

```bash
find . -exec file {} \; | grep -i elf | cut -d":" -f1 | xargs strip
```

### 崩溃收集

#### 离线收集

若用户环境不联网，我们只能在本机生成后由用户通过其他方式提供给我们。由于ubuntu默认不会生成coredump文件，因此需要开启。这里直接使用systemd提供的systemd-coredump工具进行安装即可。其会自动在应用崩溃后帮我们生成对应的coredump文件并保存。发生崩溃时，让用户提供位于`/var/lib/systemd/cordump`目录下的文件即可。

> $ ./build/RelWithDebInfo/examples/Demo/Demo
> ...
> [1]    132013 segmentation fault (core dumped)  ./build/RelWithDebInfo/examples/Demo/Demo
> $ coredumpctl list
> TIME                           PID  UID  GID SIG     COREFILE EXE                                                                               SIZE
> Thu 2024-08-08 16:44:59 CST 132013 1000 1000 SIGSEGV present  /home/xxx/code/Demo/build/RelWithDebInfo/examples/Demo/Demo 560.9K
> $ ls /var/lib/systemd/coredump/
> core.Demo.1000.8cbcac4231e94f4ca88c9a3bf8a36948.132013.1723106699000000.zst

#### 在线收集

在线收集目指的是在崩溃后应用自动将崩溃文件上传到指定服务器，目前业内主流的方案是接入[sentry](https://sentry.io/welcome/)。其提供了对dump自动生成、上传、分析和管理的能力，是一套完善的解决方案，十分强大，但该服务是收费的。官方提供了各个语言和框架的接入SDK和文档。对于C++项目，使用以下conan包[sentry-native/0.7.6](https://conan.io/center/recipes/sentry-native?version=0.7.6)，按照官方文档接入即可。

当然，我们也可以自建服务器，然后基于google提供的crashpad等库实现崩溃手机和上传功能，相关的文档网上非常多，这里就不多赘述了。
