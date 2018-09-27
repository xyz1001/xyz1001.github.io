---
title: 软件折腾笔记
author: 张帆
tags:
  - 软件折腾笔记
abbrlink: 24426
date: 2018-01-01 16:17:57
---

本人平时比较喜欢发掘和尝试各种能够为自己带来便利或乐趣的软件，但折腾的软件一多，或着部分仅在特定情况下才会去使用的软件，时间一长也便忘了如何使用，又要重新折腾一遍，因此通过《软件折腾笔记》系列进行一个记录。本文用于此系列文章的目录汇总。
<!--more-->

## 说明

本文在各软件条目后会通过一些图标标识该软件的一些属性，图标含义如下表。

| 图标                                                                    | 含义        |
| ---                                                                     | ---         |
| ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png)     | Linux软件   |
| ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) | Windows软件 |
| ![android](http://onyvldqhl.bkt.clouddn.com/software-note/android.png) | Android软件 |
| ![web](http://onyvldqhl.bkt.clouddn.com/software-note/web.png)         | 网页软件    |
| ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png)       | 免费软件    |
| ![nonfree](http://onyvldqhl.bkt.clouddn.com/software-note/nonfree.png) | 付费软件    |
| ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)   | 开源软件    |
| ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)   | 命令行软件    |

## 目录

- [WordPress](http://xyz1001.xyz/articles/10685.html) - 博客系统 ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)
 一款基于PHP开发的动态博客系统，具有丰富的主题、插件。

- [hexo](http://xyz1001.xyz/articles/17074.html) - 静态博客系统 ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 一款基于nodejs开发的静态博客系统，具有丰富的主题和插件。

- [shadowsocks](http://xyz1001.xyz/articles/16675.html) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![android](http://onyvldqhl.bkt.clouddn.com/software-note/android.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)

- [aria2](https://github.com/aria2/aria2) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 一个强大的下载器后端，可以和众多前端工具配合使用，如uGet，浏览器插件甚至是网页等。[相关配置文件](https://github.com/xyz1001/dotfiles/tree/master/.aria2)

- [axel](https://github.com/axel-download-accelerator/axel) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 命令行下载器，支持多线程下载，可通过命令`axel -n [线程数] -a [百度网盘下载连接]`提高百度网盘下载速度。

- [BaiduPCS-go](https://github.com/iikira/BaiduPCS-Go) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 使用Golang开发的第三方命令行百度网盘，最大的特点是使用了类似命令行下文件操作的方式对百度网盘进行管理。

- [bat](https://github.com/sharkdp/bat) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 `cat`命令的升级版，支持输出语法高亮和Git集成。可使用`alias cat="bat"`替换`cat`命令，对于需要使用原始的`cat`命令时，使用`\cat`取消alias。类似的还有[ccat](https://github.com/jingweno/ccat)，`ccat`更加轻量级，[详细对比](https://github.com/sharkdp/bat/blob/master/doc/alternatives.md)

- [besttrace](https://www.ipip.net/product/client.html) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 `traceroute`命令的升级版，支持显示ip所在地。

- [bpython](https://github.com/bpython/bpython) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
    默认Python解释器的升级版，一个具有强大的代码提示和补全，语法高亮功能的python解释器。

- chrome-remote-desktop ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)
 Chrome的远程桌面，PC全平台可用，效果出色。

- [cloc](https://github.com/AlDanial/cloc) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 命令行下的代码行数统计工具。

- [copyq](https://hluk.github.io/CopyQ/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)
 PC全平台可用的剪切板管理工具，类似于Windows下的`Ditto`。特色是支持Vim按键操作。

- [doxygen](http://www.doxygen.nl/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 代码文档生成工具

- [eslint](https://eslint.org/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 Javascript lint工具，可配合其他前端工具使用，如`Vim`的[ale](https://github.com/w0rp/ale)插件
 
- [evtest](http://manpages.ubuntu.com/manpages/trusty/man1/evtest.1.html)  ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 Linux下监听和查询事件输入（如按键，鼠标，触摸等）的工具

- [gitg](https://wiki.gnome.org/Apps/Gitg/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)
 Linux下的Git图形化操作软件，平时用来看看分支图还是很好用的。

- [diff-so-fancy](https://github.com/so-fancy/diff-so-fancy) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 `git diff`命令的升级版，提供了更加易读的方式呈现文件对比信息，可通过[配置](https://github.com/xyz1001/dotfiles/blob/master/gitconfig)替换默认的`git diff`命令。

 ``` ini
 [core]
     pager = diff-so-fancy | less --tabs=4 -RFX
 [color]
     ui = true
 [color "diff-highlight"]
     oldNormal = red bold
     oldHighlight = red bold 52
     newNormal = green bold
     newHighlight = green bold 22
 [color "diff"]
     meta = yellow
     frag = magenta bold
     commit = yellow bold
     old = red bold
     new = green bold
     whitespace = red reverse
 ```

- [graphicsmagick](http://www.graphicsmagick.org/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 命令行下强大的图片处理工具，是一系列工具软件的集合，以下是其中部分工具：

 - convert
  图片格式转换工具
 - identity
  图片信息读取工具
 - mogrify
  添加图片效果工具
 - composite
  图片合成工具

- [highlight](http://www.andre-simon.de/doku/highlight/highlight.php) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 将文件渲染成带语法高亮的HTML

- [jq](https://stedolan.github.io/jq/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 命令行下的`json`格式化工具，Vim中可使用命令`:%!jq '.'`格式化json文件

- [kid3-qt](https://kid3.sourceforge.io/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![android](http://onyvldqhl.bkt.clouddn.com/software-note/android.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)
 音乐文件id3标签编辑工具，可用于解决音乐文件在播放器中信息显示乱码

- [meld](http://meldmerge.org/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)
 PC全平台可用的文件对比工具，`diff`命令的GUI版本。

- mlocate ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 本地文件搜索工具，通过提前生成索引加快文件搜索速度。

- [mosh](https://mosh.org/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 具有更高的稳定性的ssh，适合在网络不稳定的情况下使用。

- [moreutils](https://joeyh.name/code/moreutils/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 一系列有用的小工具的集合。下面是部分工具：

 - chronic
  以安静的方式执行一个命令，仅输出错误信息
 - errno
  查询errno的信息
 - ifdata
  查询网络相关信息而无需解析`ifconfig`命令的输出，适合用于脚本。
 - ifno
  一般用在管道里，仅当标准输入不为空时才执行后面的命令，避免一些命令在标准输入为空时报错，如`find . -name "null" | xargs rm`，在没有找到文件时rm命令会报错，修改为`find . -name "null" | ifne xargs rm`则不会
 - isutf8
  用于判断一个文件或标准输入是否是`utf-8`编码
 - lckdo
  命令执行的互斥锁，将来会被`flock`替换
 - mispipe
  执行两条命令通过管道组成的命令，但将第一条命令的返回值作为整个命令的返回值。
 - parallel
  并行执行命令
 - pee
  将标准输入作为管道传递给多个其他命令，类比于`tee`，`tee`是将标准输入传递给文件。
 - sponge
  在完整接收标准输入后再将标准输入写入文件，避免类似与`sort file > file`导致的文件被清空的问题，我们可以使用`sort file | sponge file`来解决这个问题。
 - ts
  在标准输入每行前加上时间戳信息，如`yes | ts`
 - vidir
  使用编辑器编辑目录下的文件名，可实现快速批量删除，重命名操作等。可搭配`find`命令使用，更加强大。
 - vipe
  将标准输入导入文件并将文件用默认编辑器打开，编辑完成后输出到终端
 - zrun
  自动解压压缩文件并将解压后的文件作为参数传递给后面的命令作为参数。参考[Can you provide an example use for `zrun`?](https://unix.stackexchange.com/questions/115894/can-you-provide-an-example-use-for-zrun)

- [mypaint](http://mypaint.org/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)
 Linux下的`画图`

- [ncdu](https://dev.yorhel.nl/ncdu) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 命令行下的磁盘分析器，支持Vim操作，清理磁盘就用这个。

- [notepadqq](https://notepadqq.com/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)
 Linux下的`Notepad++`

- [privoxy](https://www.privoxy.org/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 用于将`socks`协议转化为`http`协议，搭配`ss`使用，用于支持一些仅支持`http`代理的软件，如`zeal`，相关配置命令：

 ``` bash
 sudo bash -c 'echo "forward-socks5 / 127.0.0.1:1080 ." >> /etc/privoxy/config'
 sudo systemctl start privoxy.service
 ```

- [proxychains-ng](https://github.com/rofl0r/proxychains-ng) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 用于强制让一些不支持设置代理的终端程序使用网络代理。可搭配`ss`使用，配置命令：

 ```
 sudo sed -i 's/socks4\s*127.0.0.1\s*9050/socks5 \t127.0.0.1 1080/g' /etc/proxychains.conf
 ```

- [ranger](https://github.com/ranger/ranger) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 终端下的文件管理器，支持Vim操作。

- [redshift](http://jonls.dk/redshift/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)
 中文名叫红移，屏幕色温调整工具，可自动获取地理位置。Windows下可使用[f.lux](https://justgetflux.com/)

- [ripgrep](https://github.com/BurntSushi/ripgrep) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 `grep`命令的升级版，速度最快的`grep`（来源于官方测试数据）。类似的还有[Ag](https://github.com/ggreer/the_silver_searcher)

- [shellcheck](https://www.shellcheck.net/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 `shell`脚本的lint工具

- [simplescreenrecorder](https://github.com/MaartenBaert/ssr) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)
 Linux下的屏幕录制工具

- [tidy](http://www.html-tidy.org/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 `HTML`和`XML`代码格式化和错误处理工具

- [tig](https://github.com/jonas/tig) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 命令行下的git历史查看工具

- [trash-cli](https://github.com/andreafrancia/trash-cli) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 命令行下的`回收站`，可避免文件被误删

- tree ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 树状的列出当前目录和文件结构

- [universal-ctags](https://github.com/universal-ctags/ctags) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 替换早已停止维护的[exuberant ctags](http://ctags.sourceforge.net/)

- [xcape](https://github.com/alols/xcape) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 Linux下的按键映射工具，可将修饰键映射为其他按键。搭配`setxkbmap`命令使用更为强大。如可通过下面的命令将`Caps`键敲击时映射为`Esc`键，作为修饰键时映射为`Ctrl`键，Vim党的福音。

 ``` bash
 setxkbmap -option ctrl:nocaps
 xcape -e 'Control_L=Escape'
 ```

- [xclip](https://github.com/astrand/xclip) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 Linux下在命令行对X11桌面环境的剪贴板进行操作的工具，一些其他工具的功能依赖该工具。

- [xdotool](https://github.com/jordansissel/xdotool) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png) ![cmd](http://onyvldqhl.bkt.clouddn.com/software-note/cmd.png)
 Linux下模拟鼠标和键盘输入的工具，可利用该工具实现一些自动化操作

- [zeal](https://zealdocs.org/) ![linux](http://onyvldqhl.bkt.clouddn.com/software-note/linux.png) ![windows](http://onyvldqhl.bkt.clouddn.com/software-note/windows.png) ![free](http://onyvldqhl.bkt.clouddn.com/software-note/free.png) ![opensoure](http://onyvldqhl.bkt.clouddn.com/software-note/oss.png)
 PC全平台的离线开发文档查看程序，类似于macOS下的`Dash`，文档资源来源于`Dash`。Linux下使用体验很棒，但Windows下很卡，易崩溃。
