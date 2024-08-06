---
title: Manjaro个性化配置
author: 张帆
tags:
  - manjaro
  - 配置
abbrlink: 1418
date: 2018-02-27 21:32:16
---

使用了Manjaro Deepin有一段时间了，从最初的Ubuntu kylin，到Ubuntu，CentOS，Deepin，直到去年发现了Manjaro这个发行版，被其相对简单的安装和强大的软件源所深深吸引。刚从Debian系切换过来还不是很习惯，相对于Deepin的开箱即用，Manjaro还是有一些需要手动去配置的地方。本文将以刚安装完成的Manjaro为前提，一步一步记录下自己的个性化配置步骤，方便以后查阅，同时也为他人提供借鉴。

注：本文所示系统为Manjaro Deepin 17.1.2，不保证在其他Linux发行版/桌面环境/版本下完全可用。请根据自身的需求谨慎参考。具体相关命令可参考[Manjaro个性化配置脚本](https://raw.githubusercontent.com/xyz1001/software-notes/master/backup_and_restore/manjaro/setup.sh)

<!--more-->

## 基础软件安装

### 调整软件源

1. 根据速度切换软件源

 ``` bash
 sudo pacman-mirrors -i -c China -b stable -a
 ```

2. 向`/etc/pacman.conf`添加archlinux源

 ```
 [archlinuxcn]
 SigLevel = Optional TrustAll
 Server = http://mirrors.ustc.edu.cn/archlinuxcn/$arch
 ```

### 恢复官方软件

1. 卸载不需要的软件

 ``` bash
 sudo pacman -R `comm -12 <(cat uninstall.lst | sort) <(pacman -Qnq | sort)`
 ```

2. 安装密钥环

 ``` bash
 sudo pacman -S gnupg archlinuxcn-keyring manjaro-keyring --needed
 ```

3. 更新软件数据库和软件

 ``` bash
 sudo pacman -Syyu
 ```

4. 安装备份的官方软件仓库中的软件，即可通过`pacman`进行安装的软件，生成已安装软件列表的方法可参考[Manjaro下备份已安装软件包](/2018/03/02/manjaro-backup-installed-packages/)

 ``` bash
 sudo pacman -S $(< pacman.lst) --needed
 ```

注：若archlinux源`SigLevel`字段设置为`TrustedOnly`，此处可能会出现`Signature from "User <email@gmail.com>" is unknown trust, installation failed`错误，目前原因未知，可能和系统镜像较旧有关，网上推荐的[解决方式](https://forum.manjaro.org/t/invalid-or-corrupted-package-pgp-signature/324/4)如下，我尝试过，依然不行

``` bash
sudo rm -r /etc/pacman.d/gnupg
sudo pacman -Sy gnupg archlinux-keyring manjaro-keyring
sudo pacman-key --init
sudo pacman-key --populate archlinux manjaro
sudo pacman-key --refresh-keys
sudo pacman -Sc
```

## 基本环境配置

### SSH key

复制`.ssh`备份文件夹至`~`并将权限修改为600

### dotfiles

`dotfiles`中保存了HOME目录下主要的`.`开头的配置文件

``` bash
git clone git@github.com:xyz1001/dotfiles.git && cd dotfiles && ./install.py
```

### ss-qt5

略

### privoxy

用于将socks5代理转换为http代理

 ``` bash
 sudo bash -c 'echo "forward-socks5 / 127.0.0.1:1080 ." >> /etc/privoxy/config' # 修改配置文件
 sudo systemctl start privoxy.service # 运行服务
 ```

### 调整HOME目录下文件夹

1. 修改中文目录名为英文，若安装时的语言选择的不是中文，此步可跳过
 修改`~/.config/user-dirs.dirs`文件如下

 ```
 XDG_DESKTOP_DIR="$HOME/Desktop"
 XDG_DOWNLOAD_DIR="$HOME/Download"
 XDG_TEMPLATES_DIR="$HOME/Templates"
 XDG_PUBLICSHARE_DIR="$HOME/Public"
 XDG_DOCUMENTS_DIR="$HOME/Documents"
 XDG_MUSIC_DIR="$HOME/Music"
 XDG_PICTURES_DIR="$HOME/Pictures"
 XDG_VIDEOS_DIR="$HOME/Videos"
 ```

2. 删除中文目录

 ``` bash
 cd ~ && rmdir 文档 公共 模板 下载 音乐 桌面 视频 && rm -r 图片
 ```

3. 创建其他所需文件夹

 ``` bash
 mkdir ~/{Project,Code,Application}
 ln -s /tmp ~/Temp
 ```

## 恢复其他软件

### 配置安装环境

1. 配置终端代理

 ``` bash
 export https_proxy=127.0.0.1:8118
 export http_proxy=127.0.0.1:8118
 ```

2. 配置Git代理，若Git配置文件`~/.gitconfig`中已配置，可跳过此步

 ``` bash
 git config --global http.proxy 'socks5://127.0.0.1:1080'
 git config --global https.proxy 'socks5://127.0.0.1:1080'
 ```

### Python软件

``` bash
sudo pip2 --proxy 127.0.0.1:8118 install youdao
sudo pip --proxy 127.0.0.1:8118 install you-get thefuck tldr
```

### npm软件

``` bash
sudo -E npm install hexo-cli -g
```

### 自行编译软件

#### dde-system-monitor

dock上的系统资源监视器,方便查看内存及网速

``` bash
cd /tmp && git clone git@github.com:xyz1001/dde-system-monitor.git
cd dde-system-monitor && mkdir build && cd build
qmake .. && make
sudo cp libdock-system-monitor.so /usr/lib/dde-dock/plugins && pkill dde-dock
```

#### XMemo

便签软件

``` bash
cd /tmp && git clone git@github.com:xyz1001/XMemo.git
cd XMemo && mkdir build && cd build
qmake ../src && make
sudo cp -r ../package/fakeroot/usr/* /usr/
sudo cp xmemo /usr/local/bin
```

#### cppiniter

c++工程初始化脚本

``` bash
cd /tmp && git clone git@github.com:xyz1001/cppiniter.git
cd cppiniter && sudo ./install.py
```

### AUR软件

#### albert

`Albert`需要单独安装，指定内核模块
yaourt -S albert

### deepinwine系列软件

该系列软件经常会安装出错，故单独安装，避免yaourt批量安装多次失败
yaourt -S deepin.com.qq.office
yaourt -S deepin.com.thunderspeed
yaourt -S deepin.com.wechat

#### 其他软件

安装Archlinux社区软件仓库中的软件

``` bash
yaourt -S $(< yaourt.lst) --needed --noconfirm
```

## 软件配置

### zsh

1. 将`zsh`设置为默认shell

 ``` bash
 chsh -s /usr/bin/zsh
 ```

2. 运行zsh，将自动安装相关插件

### tmux

1. 在控制中心中添加开启终端时启动tmux快捷键`Ctrl+Alt+T`，覆盖默认设置启动命令为`deepin-terminal -e tmux -2`
2. 安装tmux插件管理器tpm

 ``` bash
 git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
 ```

3. 运行tmux，按下快捷键`Ctrl+a`，`I`安装tmux插件

### vim

1. 运行vim，自动安装插件
2. 编译`Youcompleteme`依赖

 ``` bash
 cd ~/.vim/plugged/YouCompleteMe/ && ./install.py --clang-completer --system-libclang
 ```

### proxychains

```
sudo sed -i "s/socks4\s*127.0.0.1\s*9050/socks5 \t127.0.0.1 1080/g" /etc/proxychains.conf
```

### Chrome

1. 以代理模式启动Chrome

 ``` bash
 google-chrome-stable --proxy-server=socks5://127.0.0.1:1080
 ```

2. 登陆Chrome进行个人数据同步

### 深度终端

- 调整主题为`solarized dark`
- 调整字体为`firacode`
- 启动时最大化
- 关闭退出提示

 ``` bash
 sudo sed -i "s/print_notify_after_script_finish.*/print_notify_after_script_finish=false/g" ~/.config/deepin/deepin-terminal/config.conf
 ```

- 关闭雷神终端的标题栏

 ``` bash
 sudo sed -i "s/show_quakewindow_tab.*/show_quakewindow_tab=false/g" ~/.config/deepin/deepin-terminal/config.conf
 ```

### 坚果云

``` bash
# 安装
sudo pacman -S gvfs python-notify2 jdk8-openjdk --needed
wget http://www.jianguoyun.com/static/exe/installer/nutstore_linux_dist_x64.tar.gz -O /tmp/nutstore_bin.tar.gz
mkdir -p ~/.nutstore/dist && tar zxf /tmp/nutstore_bin.tar.gz -C ~/.nutstore/dist
~/.nutstore/dist/bin/install_core.sh
sed -i "s/env python/env python2/g" "${HOME}/.nutstore/dist/bin/nutstore-pydaemon.py" # 坚果云仅支持python2
sed -i "s/enterprise=false/enterprise=true/g" "${HOME}/.nutstore/dist/conf/nutstore.properties" # 开启企业模式
```

### StarUML

``` bash
# 破解
sudo sed -i '/var pk, decrypted/a\        return {\n            name: "0xcc",\n            product: "StarUML",\n            licenseType: "vip",\n            quantity: "www.qq.com",\n            licenseKey: "later equals never!"\n        };' /opt/staruml/www/license/node/LicenseManagerDomain.js
```

## 其他配置

### 控制中心相关设置

- 更换头像
- 亮度中开启`自动调节色温`
- 默认程序
- 标准字体修改为`Noto Sans CJK SC`，等宽字体修改为`Fira Code`
- 更换图标主题
- 关闭音效
- 开启时间`自动同步`
- 开启`插入鼠标禁用触摸板`

### 其他软件设置

- albert
- notepadqq
- fcitx
- Android Studio
- Zeal
- StarUML

<script src="https://utteranc.es/client.js"
        repo="xyz1001/xyz1001.github.io"
        issue-term="title"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
