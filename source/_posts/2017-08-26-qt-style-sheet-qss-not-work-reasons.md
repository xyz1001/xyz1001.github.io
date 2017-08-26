---
title: qt style sheet(QSS)无效原因整理
author: 张帆
tags:
  - qt
  - qss
date: 2017-08-26 17:52:27
---

`Qt Style Sheet`(以下简称QSS)是Qt基于`CSS2`提供的一种快速调整程序界面的方法。但在实际使用过程中，经常会遇到设置`QSS`无效的情况。本文列举了几种比较常见的原因。
<!--more-->
# 继承于`QWidget`却未重写`paintEvent(QPaintEvent *e)`函数
在：
