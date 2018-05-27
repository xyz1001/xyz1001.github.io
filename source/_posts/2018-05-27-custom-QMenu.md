---
title: 自定义QMenu样式
author: 张帆
tags:
  - Qt
  - QMenu
  - QSS
date: 2018-05-27 21:47:49
---

最近工作中需要实现一个自定义外观的菜单，但在网上搜索后发现很少有QMenu的样式自定义相关的深入解析。请教了公司的一位前辈，他提到QMenu自定义样式不方便，于是他一般是自己实现一个菜单控件。但这样未免太过于麻烦，因此经过一番摸索后基本实现了自己所需的样式。

<!--more-->

## QMenu的子部件布局

使用过QSS(Qt Style Sheet)自定义过比较复杂的控件，如QSlider等一般都知道Qt中的控件包含一到多个subcontrol(下文翻译为子控件)。Qt自带的控件所包含的子控件可以在[Qt stylesheet reference](http://doc.qt.io/qt-5/stylesheet-reference.html)上查阅。但文档上并没有给出子控件间的相对关系。
经查阅文档中的QMenu一栏，我们可以知道QMenu包含`item`, `indicator`,`separator`,`right-arrow`,`scroller`,`tearoff`,相对来说属于子控件比较多的控件了。为了确定子控件的相对位置关系，我们可以通过对子控件设置不同的背景色和边框来进行查看。效果如图所示。
>  TODO:  <27-05-18, yourname> > 

