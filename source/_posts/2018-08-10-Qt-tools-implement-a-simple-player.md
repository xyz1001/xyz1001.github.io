---
title: Qt从0到1之工具篇：实现一个简陋的播放器
author: 张帆
tags:
  - Qt
  - Qt工具
  - Qt从0到1
abbrlink: 16527
date: 2018-08-10 21:52:35
---

[示例代码](https://github.com/xyz1001/QtExamples/tree/master/QtPlayer)

Qt提供了相当多的工具方便我们进行开发，但其中有部分工具不便于单独讲解，因此本文将通过实现一个简单的播放器来介绍如何使用这些工具。本文将通过两种途径来完成软件的开发，一种使用Qt的图形化（`GUI`）工具来完成，另一种主要使用命令行（`CLI`）工具，两种途径不是绝对独立的，完全可以在不同的步骤中使用命令行或图形化工具，这需要看个人的使用习惯。由于本文重点是让大家对Qt的工具在开发过程中的作用有一个初步了解，因此每一步的具体细节将不会展开，大家可以查阅相关文档深入了解。

我们要实现的播放器主要功能是打开并播放选择的视频文件，并提供暂停和停止播放功能，程序的效果如下图所示
![播放器效果图](https://blog-1251989759.cos.ap-guangzhou.myqcloud.com/blog/simple_player/qtplayer.png)
为了实现这个播放器，我们将使用到一个第三方库[VLC-Qt](https://github.com/vlc-qt/vlc-qt)，这个库可以用来播放多种视频格式，性能优秀，相对于Qt自带的`QMultiMedia`模块要强大很多。这个库需要手动编译，我们可以参考[官方文档](https://github.com/vlc-qt/vlc-qt/blob/master/BUILDING.md)进行编译，Windows和Macos下我们也可以直接[下载](https://github.com/vlc-qt/vlc-qt/releases)已编译好的动态库，这里不再赘述。

<!--more-->

## 使用GUI工具

### 新建工程

1. 运行`Qt Creator`，在欢迎界面(`Welcome`) -> 项目(`Projects`)中选择新建项目(`New Project`)，进入新建向导。
2. 选择项目类型为`Application` -> `Qt Widget Application`，点击确定，进入`Qt Widget Application`项目配置。
3. 输入项目名并选择项目文件夹目录。这里项目命名为QtPlayer-GUI
 对于常用的目录，我们可以选择`设为默认项目目录(Use as a default project location)`，避免重复选择。
4. 选择构建套件。这里会列出我们在设置中添加的所有可用的构建套件，我们需要根据软件的需求和外部条件来选择合适的套件，如为了更好地兼容性，我们可以选择32位的构建套件。当然我们可以选择多个构建套件，在构建时切换。若我们选错了构建套件，最简单的解决办法是关闭`Qt Creator`，删除`Qt Creator`生成的`.pro.user`文件夹，然后再用`Qt Creator`打开`.pro`文件，这样会让我们重新选择构建套件。这里我们需要选择和编译`VLC-Qt`时一致的构建套件环境，避免链接时出错。
5. 设置主界面类。这里我们可以选择主界面类的基类类型，设置类名和文件名，并选择是否添加对应的UI文件。这里我们将基类设为`QWidget`，其他保持保持默认即可。
6. 信息总览。直接点击确定，我们可以看到，`Qt Creator`自动帮我们生成了一些基本的代码。

### 引入第三方库

1. 右键项目管理窗口中项目名 -> 选择`添加库...(Add Library...)`，弹出库添加向导。
2. 选择`外部库(External Library)`，点击下一步
3. 选择`VLC-Qt`库的库文件和头文件目录，这里我们需要用到`VLCQtCore`和`VLCQtWidgets`两个库，我们需要添加两次。

添加完第三方库后，我们可以打开`.pro`文件，可以看到`Qt Creator`中自动添加了第三方库的相关配置，无需我们手动编写。

### 设置UI布局

双击生成的`.ui`文件，进入内置的`Qt Designer`界面，我们拖出我们需要的UI界面。如下图所示。
![UI布局](https://blog-1251989759.cos.ap-guangzhou.myqcloud.com/blog/simple_player/ui_layout.png)

### 编写代码

参考`VLC-Qt`提供的[examples](https://github.com/vlc-qt/examples/tree/master/simple-player)，我们可以很方便地实现相关功能，这部分不是重点，这里仅贴出相关代码，不过多解释。需要注意的是在Windows下，若直接下载的预编译好的`VLC-Qt`的库，推荐编译release版本，因为当前（2018.08.11）预编译版本（`VLC-Qt 1.1.0`）依赖`Visual Studio 2013`（`VC12`）的运行库（？？？大坑），Debug版本会找不到相关运行库。

``` cpp
// widget.h
#ifndef WIDGET_H
#define WIDGET_H

#include <QWidget>

#include "VLCQtCore/Instance.h"
#include "VLCQtCore/Media.h"
#include "VLCQtCore/MediaPlayer.h"
#include "VLCQtWidgets/WidgetVideo.h"

namespace Ui {
class Widget;
}

class Widget : public QWidget {
    Q_OBJECT

public:
    explicit Widget(QWidget *parent = nullptr);
    ~Widget();

private:
    void Open();
    void Pause();
    void Play();
    void Stop();

private:
    VlcInstance *instance_;
    VlcMedia *media_;
    VlcMediaPlayer *player_;
    VlcWidgetVideo *video_widget_;

private:
    Ui::Widget *ui;
};

#endif  // WIDGET_H
```

``` cpp
// widget.cpp
#include "widget.h"
#include "ui_widget.h"

#include <QFileDialog>
#include <QToolButton>

#include "VLCQtCore/Common.h"

Widget::Widget(QWidget *parent) : QWidget(parent), ui(new Ui::Widget) {
    ui->setupUi(this);
    setWindowTitle(tr("QtPlayer"));

    ui->open_btn_->setIcon(QIcon(":/open.png"));

    ui->pause_btn_->setCheckable(true);
    ui->pause_btn_->setChecked(false);
    ui->pause_btn_->setIcon(QIcon(":/pause.png"));
    ui->pause_btn_->setEnabled(false);

    ui->stop_btn_->setIcon(QIcon(":/stop.png"));
    ui->stop_btn_->setEnabled(false);

    video_widget_ = new VlcWidgetVideo(ui->player_);
    video_widget_->resize(ui->player_->size());

    instance_ = new VlcInstance(VlcCommon::args(), this);
    player_ = new VlcMediaPlayer(instance_);
    player_->setVideoWidget(video_widget_);

    connect(ui->open_btn_, &QToolButton::clicked, this, &Widget::Open);
    connect(ui->pause_btn_, &QToolButton::clicked, [this](bool checked) {
        if (checked) {
            ui->pause_btn_->setIcon(QIcon(":/play.png"));
            Pause();
        } else {
            ui->pause_btn_->setIcon(QIcon(":/pause.png"));
            Play();
        }
    });
    connect(ui->stop_btn_, &QToolButton::clicked, this, &Widget::Stop);
}

Widget::~Widget() {
    delete ui;
}

void Widget::Open() {
    QString file = QFileDialog::getOpenFileName(this, tr("Open file"),
                                                QDir::homePath(),
                                                tr("Video(*.mp4 *.avi *.mkv)"));
    if (file.isEmpty()) {
        return;
    }

    ui->open_btn_->setEnabled(false);
    ui->pause_btn_->setEnabled(true);
    ui->stop_btn_->setEnabled(true);

    media_ = new VlcMedia(file, true, instance_);
    player_->open(media_);
}

void Widget::Pause() {
    player_->pause();
}

void Widget::Play() {
    player_->play();
}

void Widget::Stop() {
    player_->stop();
    delete media_;
    media_ = nullptr;

    ui->open_btn_->setEnabled(true);
    ui->pause_btn_->setChecked(false);
    ui->pause_btn_->setIcon(QIcon(":/pause.png"));
    ui->pause_btn_->setEnabled(false);
    ui->stop_btn_->setEnabled(false);
}
```

### 安装必要文件

编译完成后，点击运行，在Windows平台下我们可能会遇到无法启动，提示找不到运行库的问题。这是因为程序运行时没有找到我们用到的`VLC-Qt`库的动态库。我们需要手动将两个`dll`文件复制到可执行文件的运行目录，这时就可以正常运行了。但手动复制文件相对比较麻烦，也不利于后期对外发布，因此我们可以利用`qmake`提供的安装功能，在构建时将依赖的动态库安装到指定目录。我们在`pro`文件中添加以下代码：

``` qmake
DESTDIR = $$PWD/out

vlc_lib.path = $$DESTDIR

win32 {
    vlc_lib.files += $$PWD/dev/bin/VLCQtCore.dll
    vlc_lib.files += $$PWD/dev/bin/VLCQtWidgets.dll
}

INSTALLS += vlc_lib
```

### 添加资源文件

运行后，我们会发现按钮上的图片没有显示，这时因为我们在代码中使用了相对路径，程序会在运行时去加载相对于 **当前工作目录**的相对路径的文件，由于我们在程序目录下没有放置图片文件，所以程序没有找到对应的图片资源，也就无法显示。我们只需要将图片复制到程序目录即可。但对于程序必要的资源文件，为了避免手动复制和不小心误删，我们可以利用Qt的资源系统来将管理这些文件。添加至资源管理系统中的文件将在编译期间编译至可执行程序中，这样就可以避免找不到文件的问题。

1. 右键项目->选择`添加新文件(Add New...)`，选择`Qt`模板下的`Qt Resource file`，输入文件名，这里我们建立一个用于管理图片资源的资源管理文件，因此命名为`image`，点击完成。
2. 我们可以发现项目管理窗口多了一项`Resource`，我们点击下方的`添加(Add)`按钮，添加一个前缀(`Add Prefix`)，修改为`/`。
3. 再点击`添加(Add)`按钮，选择`添加文件(Add Files)`，将我们需要的图片添加至资源管理系统。
4. 修改代码中的相对路径，替换为资源管理系统中对应文件的路径。

 ``` cpp
 ui->open_btn_->setIcon(QIcon(":/open.png"));
 ui->pause_btn_->setIcon(QIcon(":/pause.png"));
 ui->pause_btn_->setIcon(QIcon(":/play.png"));
 ui->stop_btn_->setIcon(QIcon(":/stop.png"));
 ```

### 国际化翻译

目前我们的程序界面上显示的都是英文，我们需要将其翻译为中文。这需要用到`Qt语言家(Qt Linguist)`。

1. 在`pro`文件中添加生成语言文件的代码，保存后执行qmake

 ```
 TRANSLATIONS += zh_CN.ts
 ```

2. 在`Qt Creator`中选择`工具(Tools)`->`外部(External)`->`语言家(Linguist)`->`更新翻译(Update Translations(lupdate))`，生成ts文件，这里面包括了我们要翻译的字符串。
3. 用`Qt Linguist`打开生成的ts文件，选择要翻译的源语言和目标语言，进行翻译。翻译完成后保存并发布，生成二进制文件`.qm`文件
4. 在`main.cpp`中添加相关代码，安装中文翻译。

 ``` cpp
 QTranslator translator;   // 需添加头文件 #include <QTranslator>
 translator.load("./zh_CN.qm");
 a.installTranslator(&translator);
 ```

对于翻译文件我们同样存在找不到文件导致翻译失败的问题，我们也可以通过资源管理系统来管理翻译后的文件。

### 打包发布

由于我们的程序运行需要依赖一系列Qt的库，因此我们在对外发布我们的程序时需要将这些动态库一并发布。但Qt的库很多，我们一个一个找我们程序以来的库比较麻烦，因此我们可以借助Qt的部署工具来寻找依赖的动态库。Windows下工具名为`windeployqt`，MacOS下叫`macdeployqt`，linux下官方暂时没有提供工具，但有一个强大的第三方工具`linuxdeployqt`。下面以Windows下为例进行说明。

1. 将编译好的可执行程序及依赖的第三方库复制到一个单独的目录
2. 打开`Qt命令行处理程序`，进入可执行程序目录，执行命令`windeployqt QtPlayer-GUI`即可

## 使用CLI工具（Linux环境）

部分步骤与上文类似，仅作简略介绍

### 新建工程

1. 创建目录`QtPlayer-CLI`并创建`main.cpp`，`player.h`，`player.cpp`三个文件
2. 执行`qmake -project .`生成`pro`文件
3. 修改`pro`文件，引入Qt模块

 ```
 QT += core gui widgets
 ```

### 引入第三方库

1. 将`VLC-Qt`的库复制到项目目录下`dev`目录
2. 修改`pro`文件，添加以下代码

 ```
 LIBS+= $$PWD/dev/lib64 -lVLCQtCore -lVLCQtWidgets
 INCLUDEPATH += $$PWD/dev/include
 ```

### 编写代码

这里我们将通过代码对控件进行布局。具体代码如下。

``` cpp
// main.cpp
#include "player.h"

#include <QApplication>

int main(int argc, char *argv[]) {
    QApplication a(argc, argv);

    Player w;
    w.show();

    return a.exec();
}
```

``` cpp
// player.h
/* Copyright (©) 2018 xyz1001 All rights reserved.
 * Author: xyz1001 <zgzf1001@gmail.com>
 *
 * -*- coding: uft-8 -*-
 */

#ifndef PLAYER_H_LDRNQIXJ
#define PLAYER_H_LDRNQIXJ

#include <QToolButton>
#include <QWidget>

#include <VLCQtCore/Instance.h>
#include <VLCQtCore/Media.h>
#include <VLCQtCore/MediaPlayer.h>
#include <VLCQtWidgets/WidgetVideo.h>

class Player : public QWidget {
    Q_OBJECT

public:
    explicit Player(QWidget *parent = nullptr);

private:
    void Open();
    void Pause();
    void Play();
    void Stop();

private:
    void InitButtons();

private:
    VlcInstance *instance_;
    VlcMedia *media_;
    VlcMediaPlayer *player_;
    VlcWidgetVideo *video_widget_;

private:
    QToolButton *open_btn_;
    QToolButton *pause_btn_;
    QToolButton *stop_btn_;
};

#endif  // PLAYER_H_LDRNQIXJ
```

``` cpp
// player.cpp
/* Copyright (©) 2018 xyz1001 All rights reserved.
 * Author: xyz1001 <zgzf1001@gmail.com>
 *
 * -*- coding: uft-8 -*-
 */

#include "player.h"

#include <QFileDialog>

#include "VLCQtCore/Common.h"

Player::Player(QWidget *parent) : QWidget(parent) {
    setWindowTitle(tr("QtPlayer"));
    resize(800, 520);

    InitButtons();

    video_widget_ = new VlcWidgetVideo(this);
    video_widget_->setGeometry(0, 0, 800, 450);

    instance_ = new VlcInstance(VlcCommon::args(), this);
    player_ = new VlcMediaPlayer(instance_);
    player_->setVideoWidget(video_widget_);

    connect(open_btn_, &QToolButton::clicked, this, &Player::Open);
    connect(pause_btn_, &QToolButton::clicked, [this](bool checked) {
        if (checked) {
            pause_btn_->setIcon(QIcon("./play.png"));
            Pause();
        } else {
            pause_btn_->setIcon(QIcon("./pause.png"));
            Play();
        }
    });
    connect(stop_btn_, &QToolButton::clicked, this, &Player::Stop);
}

void Player::Open() {
    QString file = QFileDialog::getOpenFileName(this, tr("Open file"),
                                                QDir::homePath(),
                                                tr("Video(*.mp4 *.avi *.mkv)"));
    if (file.isEmpty()) {
        return;
    }

    open_btn_->setEnabled(false);
    pause_btn_->setEnabled(true);
    stop_btn_->setEnabled(true);

    media_ = new VlcMedia(file, true, instance_);
    player_->open(media_);
}

void Player::Pause() {
    player_->pause();
}

void Player::Play() {
    player_->play();
}

void Player::Stop() {
    player_->stop();
    delete media_;
    media_ = nullptr;

    open_btn_->setEnabled(true);
    pause_btn_->setChecked(false);
    pause_btn_->setIcon(QIcon("./pause.png"));
    pause_btn_->setEnabled(false);
    stop_btn_->setEnabled(false);
}

void Player::InitButtons() {
    open_btn_ = new QToolButton(this);
    open_btn_->setGeometry(300, 460, 50, 50);
    open_btn_->setIcon(QIcon("./open.png"));

    pause_btn_ = new QToolButton(this);
    pause_btn_->setGeometry(375, 460, 50, 50);
    pause_btn_->setCheckable(true);
    pause_btn_->setChecked(false);
    pause_btn_->setIcon(QIcon("./pause.png"));
    pause_btn_->setEnabled(false);

    stop_btn_ = new QToolButton(this);
    stop_btn_->setGeometry(450, 460, 50, 50);
    stop_btn_->setIcon(QIcon("./stop.png"));
    stop_btn_->setEnabled(false);
}
```

### 编译

创建`build`目录并进入，执行`qmake .. && make`，生成可执行文件。

### 配置动态库路径

编译完成后我们执行编译出的可执行程序，发现无法运行，提示缺少`VLC-Qt`的运行库，我们需要设置运行库路径，在`build`目录下执行命令

``` bash
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:../dev/lib64
```

### 添加资源文件

和使用GUI时情况一样，运行后按钮并没有图片，我们可以使用Qt提供的`rcc`命令来添加资源管理系统。

1. 创建`image`文件夹，将图片文件放入，便于管理
2. 执行命令`rcc --project . -o image.qrc`，生成资源管理文件
3. 修改`pro`文件，将生成的资源管理文件添加至`RESOURCES`变量中

 ```
 RESOURCES += $$PWD/image/image.qrc
 ```

4. 修改代码中的图片路径，替换为`qrc`文件中的路径

 ``` cpp
 ui->open_btn_->setIcon(QIcon(":./open.png"));
 ui->pause_btn_->setIcon(QIcon(":./pause.png"));
 ui->pause_btn_->setIcon(QIcon(":./play.png"));
 ui->stop_btn_->setIcon(QIcon(":./stop.png"));
 ```

### 国际化翻译

1. 在`QtPlayer-CLI.pro`文件中添加`TRANSLATIONS += zh_CN.ts`
2. 执行`lupdate ./QtPlayer-CLI.pro`命令，生成`ts`文件
3. 如果我们熟悉`ts`文件格式，可以手动去修改`ts`文件，这里我们还是调用Qt语言家(`linguist`)进行翻译。执行命令`linguist zh_CN.ts`，后续步骤同使用GUI工具中的[国际化翻译](#国际化翻译)的3，4步

### 打包发布

linux下，我们可以使用第三方工具[linuxdeployqt](https://github.com/probonopd/linuxdeployqt)来进行打包，该工具可以帮助我们生成[Appimage](https://appimage.org/)文件格式。具体用法可参考项目的README。
