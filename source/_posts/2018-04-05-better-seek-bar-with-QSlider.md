---
title: Qt自定义控件之SeekBar
author: 张帆
tags:
  - Qt
  - QSlider
date: 2018-04-05 14:27:08
---

在客户端软件开发中，`SeekBar`(拖动条)是一种常见的控件，经常用于播放器，控制面板等窗口中。Qt默认提供了`QSlider`来提供相应的功能，然而`QSlider`和我们常见(如安卓中的[SeekBar](http://www.zoftino.com/android-seekbar-and-custom-seekbar-examples))的`SeekBar`相比，有几处不同

1. 无法点击跳转，`QSlider`默认行为是点击后向点击处移动[特定长度](http://doc.qt.io/qt-5/qabstractslider.html#pageStep-prop)
2. 滑块中心无法移动到两端端点
3. 存在信号冗余(调用`setValue`方法后会发出`valueChanged`信号，往往会带来问题)

<!--more-->

为了解决以上问题，我们需要继承`QSlider`，实现自己的`SeekBar`类。实现效果如下
![SeekBar](http://onyvldqhl.bkt.clouddn.com/seekbar/seekbar.png)

## 使用QSS进行修饰美化

`QSlider`是一个QSS比较复杂的控件，其Subcontrol较多，具体可参考[QSS Subcontrol#QSlider](https://qtdebug.com/qtbook-qss-subcontrol/#QSlider)。简单来说，`QSlider`下包括一个滑槽(`groove`)，`groove`中又包括已滑过区域(`sub-page`)，未滑过区域(`add-page`)和滑块(`handle`)三个部分。下文中`sub-page`和`add-page`将统称为进度条。`SeekBar`的QSS如下

``` css
QSlider {
    height: %1px; /* SeekBar的高度 */
    background: transparent;
}

QSlider::groove:horizontal {
    background: transparent;
    height: %1px; /* 滑槽与控件高度一致 */
}

QSlider::sub-page:horizontal {
    height: %2px; /* 已滑过区域的高度，这里设置的是SeekBar高度的1/5 */
    background-color: #0094FF; /* 蓝色 */
    margin: %3px 0px %3px 0px; /* 将上下多余空白用margin填充，否则会被拉伸导致height属性无效 */
    border-radius: %4px; /* 划过区域高度的一半，实现圆角效果 */
    margin-left:%5px; /* 左侧留白1/2 滑块宽度，实现滑块在左端时中心位于进度条左端点 */
}

QSlider::add-page:horizontal {
    height: %2px; /* 未滑过区域的高度，这里设置的是SeekBar高度的1/5 */
    background-color: #E6FFFFFF;
    margin: %3px 0px %3px 0px;
    border-radius: %4px;
    margin-left:%5px; /* 右侧留白1/2 滑块宽度，实现滑块在右端时中心位于进度条右端点 */
}

QSlider::handle:horizontal {
    height: %1px; /* 滑块宽度与控件高度一致 */
    width: %1px; /* 滑块为圆形 */
    background-color: #FFFFFF; /* 未按下时显示为白色 */
    border-radius: %5px; /* 高度的一半，实现圆形效果 */
}

QSlider::handle:pressed:horizontal {
    background-color: #0094FF; /* 按下时为蓝色 */
}
```

为了提高灵活性，我们将QSS中的具体数值暂时使用`QString`的占位符代替，以便在加载中再根据`SeekBar`的高度指定，也方便在运行期间`SeekBar`的大小改变时进行调整。

``` c++
void SeekBar::LoadStyleSheet() {
    QFile file(":/seekbar.qss");
    if (file.open(QFile::ReadOnly)) {
        QString style_sheet = QString::fromLatin1(file.readAll())
                                      .arg(height())
                                      .arg(height() * 0.2)
                                      .arg(height() * 0.8 / 2)
                                      .arg(height() * 0.1)
                                      .arg(height() / 2);
        setStyleSheet(style_sheet);
        file.close();
    }
}

void SeekBar::resizeEvent(QResizeEvent *event) {
    LoadStyleSheet();
    QWidget::resizeEvent(event);
}
```


## 屏蔽setValue()时的信号

由于在大多数情况下，`QSlider`只是作为一个进度指示的控件，我们往往会监听`QSlider::valueChanged`信号做出相应的跳转动作，也会在进度改变时使用`QSlider::setValue()`设置值，这就导致我们可能会在设置值后收到`valueChanged`信号并在做出相应处理后又调用`setValue`，从而导致死循环。如在一个播放器中，我们可能会监听`QSlider::valueChanged`信号，在用户手动调整进度时调用`QMediaPlayer::setPosition`调整播放进度，而且会在`QMediaPlayer`发出`positionChanged`信号后调用`QSlider::setValue`。一旦这样，就会发生以下循环：
![死循环](http://onyvldqhl.bkt.clouddn.com/seekbar/seekbar_circle.png)
为了避免这个死循环，我们在调用`QSlider::setValue`时需要屏蔽`QSlider::valueChanged`信号。因此我们重写该方法。

``` c++
void SeekBar::setValue(int value) {
    blockSignals(true);
    QSlider::setValue(value);
    blockSignals(false);
}
```

## 实现点击跳转

该效果的实现方法在网上有很多，如[QSlider mouse direct jump](https://stackoverflow.com/questions/11132597/qslider-mouse-direct-jump),然而如果直接使用回答中的方法，会发现存在一些问题。

1. 由于我们在QSS中为进度条左右各设置了宽度为1/2滑块高度的`margin`，进度条的的左端点并非控件的最左端，进度条的宽度也并非控件的宽度，导致当我们点击进度条左右端点时，预期应该设置为最小/最大值，然而实际上却并非如此。因此，我们在通过位置计算比例确定值时，需要进行一些调整。
2. 滑块拖动释放后也会触发，触发了两次`valueChanged`信号，在存在误差的情况下会导致细微的抖动
经过修正后的代码如下

``` c++
void SeekBar::mouseReleaseEvent(QMouseEvent *event) {
    if (isSliderDown() && orientation() == Qt::Horizontal) { // 判断滑块是否被按下
        // 此处由于左右margin被设置为滑块半径，即控件高度一半，因此使用height()数值进行调整，需根据实际设置调整
        QSlider::setValue(minimum() + ((maximum() - minimum()) *
                                       (event->x() - height() / 2)) /
                                              (width() - height()));
    }
    QSlider::mouseReleaseEvent(event);
}
```

## 总结

经过以上调整后，`SeekBar`已基本满足日常使用，用户可根据具体设计再做调整。[完整代码](https://github.com/xyz1001/qt-utils/tree/develop/src/widgets/seekbar)
