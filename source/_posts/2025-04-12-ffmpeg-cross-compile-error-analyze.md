---
title: FFmpeg 交叉编译问题根因分析
date: 2025-04-12 15:38:53
categories:
  - 软件工程
tags:
  - FFmpeg 
  - 交叉编译
---

# 背景

在内部一个嵌入式平台开发项目中，我们需要将 FFmpeg 交叉编译，用于音视频解码等。我们的使用方式是基于 `Conan` 打包了对应平台的交叉编译工具链（类似 [NDK](https://conan.io/center/recipes/android-ndk?version=r27c)），并通过 `Conan` 依赖 `ffmpeg/5.1.3`。在交叉编译时遇到了以下报错：

```bash
./config.h:18:19: error: expected identifier or '(' before 'void'
   18 | #define getenv(x) NULL
      |                   ^~~~
./config.h:18:19: error: expected ')' before numeric constant
   18 | #define getenv(x) NULL
      |                   ^~~~
... 
src/libavutil/libm.h:54:32: error: static declaration of 'cbrt' follows non-static declaration
   54 | static av_always_inline double cbrt(double x)
      |                                ^~~~
... 
src/libavutil/libm.h:465:40: error: static declaration of 'truncf' follows non-static declaration
  465 | static av_always_inline av_const float truncf(float x)
      |                                        ^~~~~~
```

报错日志非常长。除了最开始的 `getenv` 报错，其他报错都是类似的日志，只是报错的方法不同。统计下来有 `cbrt`、`cbrtf`、`erf`、`hypot`、`lrint`、`lrintf`、`rint`、`round`、`roundf`、`trunc`、`truncf` 这些方法。这些方法的特点是都是数学处理相关的函数。

<!--more-->

# 分析

## 网络搜索

由于 FFmpeg 构建使用的是 `Autotools`，且项目很庞大，因此直接读源码是不太合适的。首先把报错的一些特征字符串通过搜索引擎搜索，发现确实有少量相关的资料，如[这篇博客](https://blog.csdn.net/try_zp_catch/article/details/111935036)和[这个 Stack Overflow 问题](https://stackoverflow.com/questions/57681336/build-ffmpeg-4-2-with-android-ndk-r20)。但基本上所有给出的方案都是手动修改构建目录下的 `config.h`，将其中对上述报错的函数对应的 `HAVE_XXX` 宏的值修改为 `1`。

```c
#define HAVE_GETENV 0
#define HAVE_CBRT 0
#define HAVE_CBRTF 0
#define HAVE_ERF 0
...
```

在我们的使用场景中，由于该文件是 `Conan` 在调用 `Autotools` 进行配置过程中生成的，除非手动修改 FFmpeg 的 `Conan` 包，否则没有修改依赖库编译中间产物的方式。而手动修改 `Conan` 包是非常不优雅的方式，会导致需要自行维护该包。和其他同事沟通，其之前在另一个平台交叉编译时也遇到了类似的问题，也是通过修改 `config.h` 来解决的。

## 深入源码

既然网上常见的方案无法解决问题，只能去看看 FFmpeg 源码是如何生成 `config.h` 的。从上述的解决方案中我们可以知道，编译环境中是存在报错的这些函数的，但 `Autotools` 检测时却认为这些函数不存在，因此使用了 FFmpeg 自行实现的一套，导致和环境中的对应函数签名发生冲突。那 `Autotools` 是怎么检测的呢？根据过往的经验，大部分项目通常是编写一小段使用了待测试函数的代码并尝试编译，若可以编译通过则认为编译环境中存在对应函数。所以我们先尝试找到 FFmpeg 检测这些函数是否可用的实现逻辑。

在 FFmpeg 的 `configure` 文件中，我们可以看到其检测 `getenv` 函数可用性是通过一个名为 `check_func_headers` 的 shell 函数实现的，而其他数学函数则是通过名为 `check_mathfunc` 的 shell 函数实现。

```bash
log(){
    echo "$@" >> $logfile
}
...
# 测试编译指令
test_cc(){
    log test_cc "$@" # 日志输出
    cat > $TMPC # 从标准输入中读取并临时文件
    log_file $TMPC # 打印临时文件内容
    test_cmd $cc $CPPFLAGS $CFLAGS "$@" $CC_C $(cc_o $TMPO) $TMPC # 调用编译器进行编译
}

# 测试编译和链接指令
test_ld(){
    log test_ld "$@"
    type=$1
    shift 1
    flags=$(filter_out '-l*|*.so' $@)
    libs=$(filter '-l*|*.so' $@)
    test_$type $($cflags_filter $flags) || return # 这里type由上层传入，在当前场景为cc，也就是会调用到上面的test_cc，进行编译
    flags=$($ldflags_filter $flags)
    libs=$($ldflags_filter $libs)
    test_cmd $ld $LDFLAGS $LDEXEFLAGS $flags $(ld_o $TMPE) $TMPO $libs $extralibs # 调用链接命令进行链接
}
...
# 测试数学处理相关方法
check_mathfunc(){
    log check_mathfunc "$@"
    func=$1
    narg=$2
    shift 2
    test $narg = 2 && args="f, g" || args="f"
    disable $func
    test_ld "cc" "$@" <<EOF && enable $func # 使用heredoc语法，生成一段代码作为标准输入并调用编译链接方法
#include <math.h>
float foo(float f, float g) { return $func($args); }
int main(void){ return (int) foo; }
EOF
}

# 测试系统标准头文件及其包含的方法
check_func_headers(){
    log check_func_headers "$@"
    headers=$1
    funcs=$2
    shift 2
    {
        for hdr in $headers; do
            print_include $hdr
        done
        echo "#include <stdint.h>"
        for func in $funcs; do
            echo "long check_$func(void) { return (long) $func; }"
        done
        echo "int main(void) { int ret = 0;"
        # LTO could optimize out the test functions without this
        for func in $funcs; do
            echo " ret |= ((intptr_t)check_$func) & 0xFFFF;"
        done
        echo "return ret; }"
    } | test_ld "cc" "$@" && enable $funcs && enable_sanitized $headers
}

...
logfile="ffbuild/config.log" # 日志路径
...
check_func_headers stdlib.h getenv
...
for func in $MATH_FUNCS; do
    eval check_mathfunc $func \${${func}_args:-1} $libm_extralibs
done
```

从上述代码和注释中我们可以很清晰地看到，FFmpeg 的 `configure` 文件对每个待测试的函数，根据函数类型不同编写了对应的临时代码，并生成临时文件进行编译和链接，根据结果确定 `HAVE_XXX` 宏的值，并写入 `config.h`。这里我们不太好确定编译链接测试代码时的参数，但可以注意到一个很重要的信息是 `configure` 脚本会将这些检测过程中的命令（包括参数）详细地写入日志文件，日志文件路径为 `ffbuild/config.log`。我们接下来可以从这个日志文件入手。

## 分析日志

在 `Conan` 的缓存目录下很容易找到这个文件，其中确实包含了报错函数的检测过程。例如 `getenv` 的检测日志如下：

```java 
check_func_headers stdlib.h getenv
test_ld cc
test_cc
BEGIN /tmp/ffconf.XZKtLanu/test.c
    1  #include <stdlib.h>
    2  #include <stdint.h>
    3  long check_getenv(void) { return (long) getenv; }
    4  int main(void) { int ret = 0;
    5   ret |= ((intptr_t)check_getenv) & 0xFFFF;
    6  return ret; }
END /tmp/ffconf.XZKtLanu/test.c
arm-pokymllib32-linux-gnueabi-gcc -D_ISOC99_SOURCE -D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE -D_POSIX_C_SOURCE=200112 -D_XOPEN_SOURCE=600 -DPIC -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security --sysroot=/home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -g -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -g -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security -std=c11 -fPIC -mthumb -c -o /tmp/ffconf.XZKtLanu/test.o /tmp/ffconf.XZKtLanu/test.c
arm-pokymllib32-linux-gnueabi-ld --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -o /tmp/ffconf.XZKtLanu/test /tmp/ffconf.XZKtLanu/test.o
arm-pokymllib32-linux-gnueabi-ld: warning: cannot find entry symbol _start; defaulting to 0000000000010094
arm-pokymllib32-linux-gnueabi-ld: /tmp/ffconf.XZKtLanu/test.o: in function `check_getenv':
/tmp/ffconf.XZKtLanu/test.c:3: undefined reference to `getenv'
```

可以清晰地看到其调用了 `check_func_headers` 这个函数，生成了一段测试代码并写入 `/tmp/ffconf.XZKtLanu/test.c` 文件中，接着调用 `GCC` 进行编译，最后调用 `LD` 进行链接。但在最后链接时报错了，提示引用了未定义的符号 `getenv`，这说明链接参数存在问题，没有链接到包含 `getenv` 符号的 `libc` 库。经过手动执行这些命令，果然能复现。所以现在最直接的思路就简单很多了：找到能让这段示例代码编译通过的方式。

通常在编译一个 `Hello World` 时，我们不会将编译和链接拆成两条命令来执行，而是直接通过类似于 `gcc main.cpp -o main` 的方式一步完成。对于这段示例代码，使用类似的命令进行编译链接（精简了部分重复参数），测试发现没有任何问题。

```bash
arm-pokymllib32-linux-gnueabi-gcc -v -D_ISOC99_SOURCE -D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE -D_POSIX_C_SOURCE=200112 -D_XOPEN_SOURCE=600 -DPIC -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security --sysroot /home/xyz1001/.conan2/p/b/xxx98db1d1e4d72ab8/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -g -std=c11 -fPIC -mthumb -c -o test test.c
```

这说明通过 GCC 间接调用 LD 是没问题的。结合[网上的说明](https://stackoverflow.com/questions/13169462/generating-executable-with-ld)，在执行 LD 时，还需要额外传入大量的参数，比如动态链接器，C 运行时等，但很明显 FFmpeg 生成的链接指令是有问题的，导致链接失败，进而误判断被测试方法不可用，最终导致实际代码编译时出错。

## 对比测试

到这里我们找到了直接原因，那为什么 FFmpeg 只会在交叉编译时生成错误的指令呢，毕竟在非交叉编译时是没有问题的。带着这个问题，我尝试在非交叉编译下编译 FFmpeg 并查看对应的构建日志

```java
check_func_headers stdlib.h getenv
test_ld cc
test_cc
BEGIN /tmp/ffconf.SmvA0ZqN/test.c
    1  #include <stdlib.h>
    2  #include <stdint.h>
    3  long check_getenv(void) { return (long) getenv; }
    4  int main(void) { int ret = 0;
    5   ret |= ((intptr_t)check_getenv) & 0xFFFF;
    6  return ret; }
END /tmp/ffconf.SmvA0ZqN/test.c
gcc -D_ISOC99_SOURCE -D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE -D_POSIX_C_SOURCE=200112 -D_XOPEN_SOURCE=600 -DPIC -m64 -g -m64 -g -std=c11 -fPIC -c -o /tmp/ffconf.SmvA0ZqN/test.o /tmp/ffconf.SmvA0ZqN/test.c
gcc -m64 -m64 -Wl,--as-needed -Wl,-z,noexecstack -o /tmp/ffconf.SmvA0ZqN/test /tmp/ffconf.SmvA0ZqN/test.o
```

神奇的是，这里 FFmpeg 同时使用 GCC 命令进行编译和链接，而不是使用 LD。由于 GCC 内部调用 LD 时会自动补全 LD 所需的额外参数，因此没有出现问题。到这里我意识到，是不是 FFmpeg 就要求链接器命令是 GCC 呢，毕竟这里的 `-Wl` 参数的作用就是直接传递给 GCC 编译器，告知其再将后面的参数传递给链接器。我们再次打开交叉编译时的构建日志，果然在日志中发现了大量类似如下的报错

```java
test_ldflags -Wl,-Bsymbolic
test_ld cc -Wl,-Bsymbolic
test_cc -Wl,-Bsymbolic
BEGIN /tmp/ffconf.XZKtLanu/test.c
    1  int main(void){ return 0; }
END /tmp/ffconf.XZKtLanu/test.c
arm-pokymllib32-linux-gnueabi-gcc -D_ISOC99_SOURCE -D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE -D_POSIX_C_SOURCE=200112 -D_XOPEN_SOURCE=600 -DPIC -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security --sysroot=/home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -g -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -g -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security -std=c11 -fPIC -mthumb -g -Wdeclaration-after-statement -Wall -Wdisabled-optimization -Wpointer-arith -Wredundant-decls -Wwrite-strings -Wtype-limits -Wundef -Wmissing-prototypes -Wstrict-prototypes -Wempty-body -Wno-parentheses -Wno-switch -Wno-format-zero-length -Wno-pointer-sign -Wno-unused-const-variable -Wno-bool-operation -Wno-char-subscripts -Wl,-Bsymbolic -c -o /tmp/ffconf.XZKtLanu/test.o /tmp/ffconf.XZKtLanu/test.c
arm-pokymllib32-linux-gnueabi-ld --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -Wl,-Bsymbolic -o /tmp/ffconf.XZKtLanu/test /tmp/ffconf.XZKtLanu/test.o
arm-pokymllib32-linux-gnueabi-ld: unrecognized option '-Wl,-Bsymbolic'
arm-pokymllib32-linux-gnueabi-ld: use the --help option for usage information
```

这段日志在检测链接参数的可用性，正常来说这里应该是可以通过的，但我们仔细看链接时传递的参数，其使用的格式是 `-Wl` + 链接参数，这种格式是 `gcc` 命令的参数，表示将 `-Wl` 后的参数转发给链接器。至此我们可以确定，FFmpeg 期望的链接器是使用 `gcc`，而不是 `ld`。

尝试在交叉编译工具链 Conan 包中，将链接器环境变量 `LD` 从 `arm-pokymllib32-linux-gnueabi-ld` 修改成 `arm-pokymllib32-linux-gnueabi-gcc`，果然顺利编过 FFmpeg。

## 小心验证

目前为止我们看上去解决了问题，但进行的修改非常反直觉，我们将 `LD` 环境变量定义为了编译器命令 `gcc`，虽然 `gcc` 也可以间接调用 `ld` 命令完成链接，[SO 上该回答](https://stackoverflow.com/a/8694291)也建议这么处理。但由于使用 `gcc` 和 `ld` 时传递的参数不一致，使用 `gcc` 链接时链接参数需要通过 `-Wl` 的方式传递。那会不会存在一个第三方库认为 `LD` 表示的是 `ld` 命令从而没有添加 `-Wl`，进而导致编译失败呢？最简单的验证方式就是直接将目前使用到的第三方库全部重新构建一边试试。果不其然，FAAC 这个库在修改 `LD` 定义前构建没有问题，但修改后编译出错了。比较关键的日志如下：

``` bash
checking for ld used by arm-pokymllib32-linux-gnueabi-gcc... arm-pokymllib32-linux-gnueabi-gcc
checking if the linker (arm-pokymllib32-linux-gnueabi-gcc) is GNU ld... no
...
/bin/sh ../libtool  --tag=CC   --mode=link arm-pokymllib32-linux-gnueabi-gcc -fvisibility=hidden  -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security --sysroot=/home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -g -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security -no-undefined  --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -o libfaac.la -rpath //lib libfaac_la-bitstream.lo libfaac_la-fft.lo libfaac_la-frame.lo libfaac_la-blockswitch.lo libfaac_la-util.lo libfaac_la-channels.lo libfaac_la-filtbank.lo libfaac_la-tns.lo libfaac_la-quantize.lo libfaac_la-huff2.lo libfaac_la-huffdata.lo libfaac_la-stereo.lo  -lm
...
/bin/sh ../libtool  --tag=CC   --mode=link arm-pokymllib32-linux-gnueabi-gcc  -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security --sysroot=/home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -g -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security  --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -o faac main.o input.o mp4write.o ../libfaac/libfaac.la -lm
libtool: link: arm-pokymllib32-linux-gnueabi-gcc -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security --sysroot=/home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -g -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -o .libs/faac main.o input.o mp4write.o  ../libfaac/.libs/libfaac.so -lm
/home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/x86_64-pokysdk-linux/usr/libexec/arm-poky-linux-gnueabi/gcc/arm-poky-linux-gnueabi/11.5.0/ld: cannot find ../libfaac/.libs/libfaac.so: No such file or directory
```

FAAC 同样也是一个 Autotools 项目，其在 configure 过程中检测了链接器，发现并不是 ld，在库的链接时，虽然我们指定了编译动态库，但却并没有生成动态库，导致最后构建 FAAC 命令行应用，链接时没找到 libfaac 的动态库。

同样我们打开 FAAC 的构建目录，发现了其也有对应的 config.log 文件。打开后搜索 ld，发现了如下的日志：

```java
configure:17066: checking for getopt_long in -lgnugetopt
configure:17089: arm-pokymllib32-linux-gnueabi-gcc -o conftest -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security --sysroot=/home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi -g -march=armv7-a -mthumb -mfpu=neon -mfloat-abi=hard -O2 -fstack-protector-strong -Wformat -Wformat-security -Werror=format-security  --sysroot /home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/armv7at2hf-neon-pokymllib32-linux-gnueabi conftest.c -lgnugetopt   >&5
/home/xyz1001/.conan2/p/b/xxx983c0ffc69ef183/p/sysroots/x86_64-pokysdk-linux/usr/libexec/arm-poky-linux-gnueabi/gcc/arm-poky-linux-gnueabi/11.5.0/ld: cannot find -lgnugetopt: No such file or directory
collect2: error: ld returned 1 exit status
```

这里 FAAC 也是在检测链接参数的可用性，但其使用的是`ld`的参数格式，没有添加`-Wl`，因此`gcc`并不认识这个参数，导致报错。这初步验证了 FAAC 是要求`LD`环境变量应当是链接器`ld`而不是`gcc`。我们再找到检测链接器的日志

``` bash
configure:5979: checking for ld used by arm-pokymllib32-linux-gnueabi-gcc
configure:6047: result: arm-pokymllib32-linux-gnueabi-gcc
configure:6054: checking if the linker (arm-pokymllib32-linux-gnueabi-gcc) is GNU ld
configure:6070: result: no
```

这里并没有输出检测的方式，我们可以根据头部的信息，找到`configure`文件的6054行

``` bash
printf %s "checking if the linker ($LD) is GNU ld... " >&6; }
if test ${lt_cv_prog_gnu_ld+y}
then :
  printf %s "(cached) " >&6
else $as_nop
  # I'd rather use --version here, but apparently some GNU lds only accept -v.
case `$LD -v 2>&1 </dev/null` in
*GNU* | *'with BFD'*)
  lt_cv_prog_gnu_ld=yes
  ;;
*)
  lt_cv_prog_gnu_ld=no
  ;;
esac
fi
{ printf "%s\n" "$as_me:${as_lineno-$LINENO}: result: $lt_cv_prog_gnu_ld" >&5
printf "%s\n" "$lt_cv_prog_gnu_ld" >&6; }
with_gnu_ld=$lt_cv_prog_gnu_ld
```

这里是通过命令 `$LD -v 2>&1 </dev/null` 的输出是否包含 `GNU` 或 `WITH BFD` 来进行判断，我们分别用 `ld` 和 `gcc` 执行，输出如下

``` bash
$ arm-pokymllib32-linux-gnueabi-gcc -v < /dev/null                                                                                           -- INSERT --
Using built-in specs.
COLLECT_GCC=arm-pokymllib32-linux-gnueabi-gcc
COLLECT_LTO_WRAPPER=/home/xyz1001/.conan2/p/b/xxx98db1d1e4d72ab8/p/sysroots/x86_64-pokysdk-linux/usr/libexec/arm-poky-linux-gnueabi/gcc/arm-poky-linux-gnueabi/11.5.0/lto-wrapper
Target: arm-poky-linux-gnueabi
Configured with: ../../../../../../work-shared/gcc-11.5.0-r0/gcc-11.5.0/configure --build=x86_64-linux --host=x86_64-pokysdk-linux --target=arm-poky-linux-gnueabi --prefix=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/usr --exec_prefix=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/usr --bindir=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/usr/bin/arm-poky-linux-gnueabi --sbindir=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/usr/bin/arm-poky-linux-gnueabi --libexecdir=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/usr/libexec/arm-poky-linux-gnueabi --datadir=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/usr/share --sysconfdir=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/etc --sharedstatedir=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/com --localstatedir=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/var --libdir=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/usr/lib/arm-poky-linux-gnueabi --includedir=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/usr/include --oldincludedir=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/usr/include --infodir=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/usr/share/info --mandir=/usr/local/oe-sdk-hardcoded-buildpath/sysroots/x86_64-pokysdk-linux/usr/share/man --disable-silent-rules --disable-dependency-tracking --with-libtool-sysroot=/home/liushenghua/build/code/build/tmp/work/x86_64-nativesdk-pokysdk-linux/gcc-cross-canadian-arm/11.5.0-r0/recipe-sysroot --with-gnu-ld --enable-shared --enable-languages=c,c++ --enable-threads=posix --enable-multilib --enable-default-pie --enable-c99 --enable-long-long --enable-symvers=gnu --enable-libstdcxx-pch --program-prefix=arm-poky-linux-gnueabi- --without-local-prefix --disable-install-libiberty --disable-libssp --enable-libitm --enable-lto --disable-bootstrap --with-system-zlib --with-linker-hash-style=gnu --enable-linker-build-id --with-ppl=no --with-cloog=no --enable-checking=release --enable-cheaders=c_global --without-isl --with-gxx-include-dir=/not/exist/usr/include/c++/11.5.0 --with-build-time-tools=/home/liushenghua/build/code/build/tmp/work/x86_64-nativesdk-pokysdk-linux/gcc-cross-canadian-arm/11.5.0-r0/recipe-sysroot-native/usr/arm-poky-linux-gnueabi/bin --with-sysroot=/not/exist --with-build-sysroot=/home/liushenghua/build/code/build/tmp/work/x86_64-nativesdk-pokysdk-linux/gcc-cross-canadian-arm/11.5.0-r0/recipe-sysroot --with-plugin-ld=ld --enable-poison-system-directories --disable-static --enable-nls --with-glibc-version=2.28 --enable-initfini-array
Thread model: posix
Supported LTO compression algorithms: zlib zstd
gcc version 11.5.0 (GCC)
 
$ arm-pokymllib32-linux-gnueabi-ld -v < /dev/null                                                                                            -- INSERT --
GNU ld (GNU Binutils) 2.38.20220708
```

因此使用 `gcc` 做链接器时没有通过，导致 `with_gnu_ld` 为 `no`，进而导致 `faac` 链接时认为当前没有可用的链接器，于是使用 `libtool` 生成了 `.la` 格式的库（不同格式的库区别可参考这个回答）。因此直接将 `LD` 环境变量修改为 `gcc` 是不可取的，会导致其他第三方库构建出错。

## LD 来源分析

到了这里，感觉似乎没有万全的解决方案了，将 `LD` 定义成哪个都有问题。那如果不定义会怎么样呢？我们再回到 FFmpeg 的源码，尝试找到其在没有定义 `LD` 时的行为。在 `configure` 文件中对应的代码如下：

```bash
...
--ld=LD                  use linker LD [$ld_default] # configure 参数，若传入 --ld 参数，则使用对应的值为 ld 命令，否则会使用 ld_default 的值
...
: ${ld_default:=$cc} # 默认为编译器命令
...
```

而在 FFmpeg 的 Conan 打包脚本中代码如下：

```python
ld = buildenv_vars.get("LD")
if ld:
    args.append(f"--ld={unix_path(self, ld)}")
```

也就是说如果不设置 `LD` 环境变量，默认就是 GCC。经过测试也是没有问题的。再看下 FAAC 的脚本。FAAC 只提供了 `configure.ac` 文件，该文件中没有和 `LD` 相关的内容。该文件经过 `autoreconf` 命令处理后就会生成 `configure` 脚本，从中可以找到如下内容：

```bash
ac_prog=ld # 默认值为 ld
if test yes = "$GCC"; then # 如果是 GCC 编译器
  # Check if gcc -print-prog-name=ld gives a path.
  { printf "%s\n" "$as_me:${as_lineno-$LINENO}: checking for ld used by $CC" >&5
printf %s "checking for ld used by $CC... " >&6; }
  case $host in
  *-*-mingw*)
    # gcc leaves a trailing carriage return, which upsets mingw
    ac_prog=`($CC -print-prog-name=ld) 2>&5 | tr -d '\015'` ;;
  *)
    ac_prog=`($CC -print-prog-name=ld) 2>&5` ;; # 执行该命令来获取链接器
  esac
...
fi
...
fi
 
LD=$lt_cv_path_LD
```

即 FAAC 构建时是通过命令 `$CC -print-prog-name=ld` 来获取链接器路径的。我们执行以下看看结果：

```bash
$ arm-pokymllib32-linux-gnueabi-gcc --print-prog-name=ld
/home/xyz1001/.conan2/p/b/xxx98db1d1e4d72ab8/p/sysroots/x86_64-pokysdk-linux/usr/libexec/arm-poky-linux-gnueabi/gcc/arm-poky-linux-gnueabi/11.5.0/ld
 
$ ls -al `arm-pokymllib32-linux-gnueabi-gcc --print-prog-name=ld`
lrwxrwxrwx xyz1001 xyz1001 67 B Tue Dec 31 16:22:30 2024  /home/xyz1001/.conan2/p/b/xxx98db1d1e4d72ab8/p/sysroots/x86_64-pokysdk-linux/usr/libexec/arm-poky-linux-gnueabi/gcc/arm-poky-linux-gnueabi/11.5.0/ld ⇒ ../../../../../bin/arm-poky-linux-gnueabi/arm-poky-linux-gnueabi-ld
```

通过该方式获取的就是真正的链接器的路径。

## 最终方案

经过以上的分析，考虑到绝大部分使用 Autotools 的项目都是通过提供 `configure.ac` 文件来实现，直接提供 `configure` 脚本的项目，如 FFmpeg，也会针对没有定义 `LD` 环境变量提供合适的默认值。因此可以尝试移除对 `LD` 环境变量的定义。经过对其他的第三方库全部重新构建测试，没有遇到问题。但该方案依然不是百分百完美的，因为不排除可能会有项目依赖 `LD` 环境变量。对于这类项目，可以考虑通过修改 Conan 脚本来进行适配。

# 总结

本文分析了如何从根本上去排查交叉编译，特别是基于 Autotools 项目时所遇到的问题的思路。一方面找到编译日志，定位问题关键点非常重要，另一方面，我们需要熟悉常见的 Shell 语法和编译链接命令，以及常见的构建系统。
