---
title: qt style sheet(QSS)无效原因整理
author: 张帆
tags:
  - Qt
  - QSS
abbrlink: 9919
date: 2017-08-26 17:52:27
---

`Qt Style Sheet`(以下简称QSS)是Qt基于`CSS2`提供的一种快速调整程序界面的方法。但在实际使用过程中，经常会遇到设置`QSS`无效的情况。本文列举了几种比较常见的原因。
<!--more-->

## 继承于`QWidget`却未重写`paintEvent(QPaintEvent *e)`函数

在QSS的[官方文档](http://doc.qt.io/qt-5/stylesheet-reference.html)中对于`QWidget`的介绍如下

> Supports only the background, background-clip and background-origin properties.
> If you subclass from QWidget, you need to provide a paintEvent for your custom QWidget as below:

``` c++
void CustomWidget::paintEvent(QPaintEvent *)
{
    QStyleOption opt;
    opt.init(this);
    QPainter p(this);
    style()->drawPrimitive(QStyle::PE_Widget, &opt, &p, this);
}
```

> The above code is a no-operation if there is no stylesheet set.

也就是说，对于一个`QWidget`，它仅仅支持设置背景色的QSS，如果一个继承于`QWidget`的子类想要支持其他的QSS，就需要像上面那样重新实现`paintEvent`函数。这是绝大部分人设置QSS无效的主要原因，StackOverflow上有不少类似的问题。

## 未正确添加`Q_OBJECT`宏

由于QSS是通过Qt的元对象系统(`The Meta-Object System`)支持，因而需要在头文件中`private`块中添加`Q_OBJECT`宏。

## QSS生效但被子控件遮挡

这一种情况在对容器类空间，如`QWidget`，`QFrame`等，对于这类控件设置QSS时尤其得注意QSS效果是否会被子部件遮挡。如下面这个例子

``` c++
Widget::Widget(QWidget *parent)
    : QFrame(parent)
{
    this->resize(100, 100);
    this->setWindowFlags(Qt::FramelessWindowHint);
    this->setStyleSheet("background: #FFFFFF; border-left: 3px solid gray;");
    btn_ = new QPushButton(this);
    btn_->resize(this->size());
}
```

这段QSS的作用是添加一个灰色左边框，然而由于`btn_`被调整为和`Widget`一样尺寸，导致左边框被按钮遮挡，效果图如下：

![左边框被遮挡](https://blog-1251989759.cos.ap-guangzhou.myqcloud.com/blog/qss_not_workborder-hidden.png)

对于这种情况，有两种解决方法，

- 将子控件进行移动至不遮挡父组件的位置
    在上述函数体内最后添加`btn_->move(3, 0);`
- 将子控件设为背景透明
    在上述函数体内最后添加`btn->setFlat(true);`

设置后效果如下

![左边框可见](https://blog-1251989759.cos.ap-guangzhou.myqcloud.com/blog/qss_not_work/border-shown.png)

**注**：

1. 对于`QPushButton`等控件设置透明的方式是将其设为扁平；
2. 图中右边和下方深色是按钮的边框。

## QSS属性顺序有误

QSS的每条属性并非是毫不相关的，很多时候某个属性的设置依赖于另一个属性的设置。

继续以上面的QSS为例，若将`background: #FFFFFF;`这条设置背景色的属性去除，后面设置边框的QSS是不会生效的。同样的，边框的宽度、样式和颜色顺序(对应于`3px solid gray`)必须固定，一旦颠倒三条中任意两条或缺少某一条，边框的QSS设置变不会生效。这一点尤其需要注意，具体的顺序可以多参考官方的示例文档。

## 总结

以上是个人遇到的几种QSS设置后无效的情况总结，实际使用中很可能还会遇到很多其他的情况导致QSS无效。QSS目前作为Qt对控件进行修饰的一种途径，具有十分灵活的特性。但QSS仍有较多的坑需要在使用中去慢慢摸索。
