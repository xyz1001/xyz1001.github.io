---
title: 自定义QMenu样式
author: 张帆
tags:
  - Qt
date: 2018-05-27 21:47:49
---

最近工作中需要实现一个自定义外观的菜单，但在网上搜索后发现很少有QMenu的样式自定义相关的深入解析。请教了公司的一位前辈，他提到QMenu自定义样式不方便，于是他一般是自己实现一个菜单控件。但这样未免太过于麻烦，因此经过一番摸索后基本实现了自己所需的样式。

<!--more-->

## QMenu的子部件布局

使用过QSS(Qt Style Sheet)自定义过比较复杂的控件，如QSlider等一般都知道Qt中的控件包含一到多个subcontrol(下文翻译为子控件)。Qt自带的控件所包含的子控件可以在[Qt stylesheet reference](http://doc.qt.io/qt-5/stylesheet-reference.html)上查阅。但文档上并没有给出子控件间的相对关系。

经查阅文档中的QMenu一栏，我们可以知道QMenu包含`item`, `indicator`,`separator`,`right-arrow`,`scroller`,`tearoff`,相对来说属于子控件比较多的控件了，其中`item`, `indicator`,`separator`,`right-arrow`是较为为常见的子控件。为了确定子控件的相对位置关系，我们可以通过对子控件设置不同的背景色和边框来进行查看。效果如图所示。
![QMenu subcontrol](qss_subcontrol.png)
根据图片，我们可以得知，QMenu由若干行组成，每一行可能是一个`::item`获`::separator`。注意到`::indicator`和`::right-arrow`位于`::item`边框内部，我们可以推出`::indicator`和`right-arrow`两个子控件包含在`::item`内。`::indicator`位于`::item`的左侧中央，`::right-arrow`位于`::item`的右侧中央。图片中`::item`的文字是和`::indicator`和`::right-arrow`重叠在一起的，因此为了避免遮挡，我们需要为`::item`设置合适的`padding-left`和`padding-right`。

## QMenu的边框阴影

QMenu的边框阴影我们可以通过设置背景图片来实现。但在设置过程中，我发现QMenu的`border-width`属性存在bug。
下图是在设置了以下QSS的效果

``` css
QMenu {
    background-color: #F00000;
    border-width: 10px 20px 30px 40px;
    border-style: solid;
    border-color: #00000F;
}
```

![wrong border width](wrong_border_width.png)

经过多次调整比较，我发现，无论四边`border-width`设置了什么样的值，实际上的边框宽度均为最后一个数值，设置的边框宽度部分会显示边框颜色，对于多出来的边框部分，则填充了背景色。这就导致我无法很好地显示边框阴影，边框阴影往往会有一定的偏移，四边的阴影宽度不会完全一致。经过一番尝试，依然无解，只好与设计师协商，提供以最宽的阴影边为边框宽度，其他边填充合适的透明部分的切图暂时避开了这个问题。

另一个问题是QMenu默认是有阴影的，我们需要去掉默认的阴影。我们可以通过为QMenu添加`Qt::NoDropShadowWindowHint`的WindowFlag解决这个问题。

## QMenu的尺寸问题

QMenu默认是会根据条目的内容动态调整自身的显示大小的，然而在实际中当我在QSS中调整了文字的字体大小后，QMenu并没有调整为合适的大小，导致文字显示不全。这个问题困扰了我很久，后来在和另一个人讨论这个问题时突然想到，是否可能是在QMenu根据内容确定大小时QSS尚未生效。由于QSS实际上是通过QStyle来实现具体效果的，而QStyle则是通过调用`QStyle::polish`方法来对控件样式做初始化。在这个方法的[文档](http://doc.qt.io/qt-5/qstyle.html#polish)中有这样一段说明：
> This function is called for every widget at some point after it has been fully created but just before it is shown for the very first time.
说明QSS生效是在控件创建完成后，这个时候控件的大小已经确定，所以我们需要在控件创建完成前设置好空间的字体属性，而不能通过QSS来设置。

## QMenu的弹出位置

QMenu默认弹出位置是和设置该菜单的控件位置相关的。但有时我们需要控制菜单的弹出位置。例如，我们需要在点击一个按钮后弹出菜单。通常情况下，我们可以通过`QPushButton::setMenu`来设置一个菜单。但这种方式QMenu固定以和按钮左对齐的方式显示，如果我们希望弹出菜单和按钮保持居中就无法实现。因此我们需要换一种方式。QMenu提供了`QMenu::exec`方法，我们可以传入`QPoint`来指定菜单弹出位置。这里有两处需要额外注意的地方。

1. QMenu其实是一个独立的顶层窗口，因此其位置是相对于整个桌面的，而不是相对于程序主窗口。在Qt的[文档](http://doc.qt.io/qt-5/qmenu.html#exec-1)中有以下说明：
 > Pops up the menu so that the action action will be at the specified global position p. To translate a widget's local coordinates into global coordinates, use QWidget::mapToGlobal().
 因此我们需要通过坐标转换来得出菜单实际弹出的位置。
2. 在计算QMenu的弹出位置时我们可能需要使用到QMenu的窗口大小属性，然而文档中提到了
 > When positioning a menu with exec() or popup(), bear in mind that you cannot rely on the menu's current size(). For performance reasons, the menu adapts its size only when necessary. So in many cases, the size before and after the show is different. Instead, use sizeHint() which calculates the proper size depending on the menu's current contents.
 所以正确的获取QMenu的窗口大小的姿势是通过`sizeHint()`方法。

## 小结

相对来说，QMenu算是一个比较复杂的控件了。对于这类控件，我们必须先仔细阅读官方的API文档，Qt的文档是相当优秀的工具，利用好这个工具可以是我们少踩很多坑。但从目前来看，Qt还存在一定的实现BUG，在实际开发中我们需要及时看清问题原因，不要在工具的BUG上花费过多时间，要善于寻找绕过BUG的途径。
