---
title: Qt从0到1之工具篇（3）：实现一个简陋的播放器
author: 张帆
tags:
  - Qt
  - vlc
date: 2018-08-10 21:52:35
---

Qt提供了相当多的工具方便我们进行开发，但其中有部分工具不便于单独讲解，因此本文将通过实现一个简单的播放器来介绍如何使用这些工具。本文将通过两种途径来完成软件的开发，一种使用Qt的图形化（`GUI`）工具来完成，另一种主要使用命令行（`CLI`）工具，两种途径不是绝对独立的，完全可以在不同的步骤中使用命令行或图形化工具，这需要看个人的使用习惯。
我们要实现的播放器主要功能是打开并播放选择的视频文件，并提供暂停和停止播放功能，程序的效果如下图所示
![播放器效果图]()
为了实现这个播放器，我们将使用到一个第三方库[`VLC-Qt`](https://github.com/vlc-qt/vlc-qt)，这个库可以用来播放多种视频格式，性能优秀，相对于Qt自带的`QMultiMedia`模块要强大很多。这个库需要手动编译，我们可以参考[官方文档](https://github.com/vlc-qt/vlc-qt/blob/master/BUILDING.md)进行编译，Windows和Macos下我们也可以直接[下载](https://github.com/vlc-qt/vlc-qt/releases)已编译好的动态库，这里不再赘述。

<!--more-->

## 使用GUI工具

### 新建工程

1. 运行`Qt Creator`，在欢迎界面(`Welcome`) -> 项目(`Projects`)中选择新建项目(`New Project`)，进入新建向导。
2. 选择项目类型为`Application` -> `Qt Widget Application`，点击确定，进入`Qt Widget Application`项目配置。
3. 输入项目名并选择项目文件夹目录。这里项目命名为QtPlayer-GUI
 对于常用的目录，我们可以选择`设为默认项目目录(Use as a default project location)`，避免重复选择。
4. 选择构建套件。这里会列出我们在设置中添加的所有可用的构建套件，我们需要根据软件的需求和外部条件来选择合适的套件，如为了更好地兼容性，我们可以选择32位的构建套件。当然我们可以选择多个构建套件，在构建时切换。若我们选错了构建套件，最简单的解决办法是关闭Qt Creator，删除Qt Creator生成的`.pro.user`文件夹，然后再用Qt creator打开`.pro`文件，这样会让我们重新选择构建套件。这里我们需要选择和编译`VLC-Qt`时一致的构建套件环境，避免链接时出错。
5. 设置主界面类。这里我们可以选择主界面类的基类类型，设置类名和文件名，并选择是否添加对应的UI文件。这里我们将基类设为`QWidget`，其他保持保持默认即可。
6. 信息总览。直接点击确定，我们可以看到，`Qt Creator`自动帮我们生成了一些基本的代码。

### 引入第三方库

1. 右键项目管理窗口中项目名 -> 选择`添加库...(Add Library...)`，弹出库添加向导。
2. 选择`外部库(External Library)`，点击下一步
3. 选择`VLC-Qt`库的库文件和头文件目录，这里我们需要用到`VLCQtCore`和`VLCQtWidgets`两个库，我们需要添加两次。

添加完第三方库后，我们可以打开`.pro`文件，可以看到`Qt Creator`中自动添加了第三方库的相关配置，无需我们手动编写。

### 设置UI布局

1. 双击生成的`.ui`文件，进入内置的`Qt Designer`界面，我们拖出我们需要的UI界面。如下图所示。
![UI布局]()

### 编写代码

参考`VLC-Qt`提供的[examples](https://github.com/vlc-qt/examples/tree/master/simple-player)，我们可以很方便地实现相关功能，这部分不是重点，这里仅贴出相关代码，不过多解释。

``` cpp
// widget.h

#ifndef WIDGET_H
#define WIDGET_H

#include <QWidget>

#include <VLCQtCore/Instance.h>
#include <VLCQtCore/MediaPlayer.h>
#include <VLCQtCore/Media.h>
#include <VLCQtWidgets/WidgetVideo.h>

namespace Ui {
class Widget;
}

class Widget : public QWidget
{
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

#endif // WIDGET_H
```

``` cpp
// widget.cpp
#include "widget.h"
#include "ui_widget.h"

#include <QToolButton>
#include <QFileDialog>

#include <VLCQtCore/Common.h>

Widget::Widget(QWidget *parent) :
    QWidget(parent),
    ui(new Ui::Widget) {
    ui->setupUi(this);
    setWindowTitle(tr("QtPlayer"));

    ui->open_btn_->setIcon(QIcon("./open.png"));

    ui->pause_btn_->setCheckable(true);
    ui->pause_btn_->setChecked(false);
    ui->pause_btn_->setIcon(QIcon("./pause.png"));
    ui->pause_btn_->setEnabled(false);

    ui->stop_btn_->setIcon(QIcon("./stop.png"));
    ui->stop_btn_->setEnabled(false);

    video_widget_ = new VlcWidgetVideo(ui->player_);
    video_widget_->resize(ui->player_->size());

    instance_ = new VlcInstance(VlcCommon::args(), this);
    player_ = new VlcMediaPlayer(instance_);
    player_->setVideoWidget(video_widget_);

    connect(ui->open_btn_, &QToolButton::clicked, this, &Widget::Open);
    connect(ui->pause_btn_, &QToolButton::clicked, [this](bool checked){
        if(checked) {
            ui->pause_btn_->setIcon(QIcon("./play.png"));
            Pause();
        } else {
            ui->pause_btn_->setIcon(QIcon("./pause.png"));
            Play();
        }
    });
    connect(ui->stop_btn_, &QToolButton::clicked, this, &Widget::Stop);
}

Widget::~Widget() {
    delete ui;
}

void Widget::Open() {
    QString file = QFileDialog::getOpenFileName(this, tr("Open file"), QDir::homePath(),tr("Video(*.mp4 *.avi *.mkv)"));
    if(file.isEmpty()) {
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
    ui->pause_btn_->setIcon(QIcon("./pause.png"));
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

## 使用CLI工具


