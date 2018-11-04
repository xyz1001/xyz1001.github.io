---
title: Qt从0到1之机制篇 - 事件
author: 张帆
tags:
  - Qt
  - Qt从0到1
  - Qt机制
abbrlink: 11422
date: 2018-08-28 17:31:45
---

[本节示例代码](https://github.com/xyz1001/QtExamples/tree/master/QtEvent)

相信大家在初学C/C++等编程语言时，编写的都是在命令行下运行的程序，也就是运行时只用一个黑框的应用程序。和我们日常使用的GUI程序相比，这些程序除了没有一个GUI界面外，还有一个很大的区别在于这些程序是一种过程式的执行方式，执行完一行代码后再去执行下一句，执行完一个函数再去执行下一个，执行完之后程序就退出了，类似于一个批处理或脚本程序，而GUI程序是可以根据内部或外部的输入来确定执行哪些代码，以什么样的顺序去执行。这些输入可能是系统的分辨率进行了修改，或者是用户点击了一个按钮，按下了键盘上的某个键，触发了一个定时器等等。程序具体如何运行，是受到这些输入控制的，如果没有任何输入，程序将不执行任何代码。这就是批处理程序设计(`batch programming`)和事件驱动程序设计(`Event-driven programming`)的一个显著差异。

在事件驱动程序设计中，事件是核心要素，一个事件就是一个来自外界的信息输入。接触过Windows编程的人应该会了解过Windows的消息循环机制，这里的消息就类似于事件。Windows程序在收到系统派发的消息后，可以根据消息类型及消息参数的不同，进行不同的处理。例如当我们按下键盘上的`Alt + F4`快捷键后，Windows就会向当前的活动窗口发送一个类型为`WM_CLOSE`的消息，这个消息一般用于通过窗口进行关闭。窗口在收到这个消息后，原则上需要将自己关闭，但也可以选择忽略这个消息，这样我们的快捷键就“失效”了。

<!--more-->

## 事件分发

由于不同的操作系统提供了不同的消息机制，消息类型差异较大，因此Qt为了更好的跨平台性，对常见的通用的消息进行了封装。Qt的程序中，绝大部分情况下我们会在`main`函数中调用`QCoreApplication::exec()`方法。这个方法就是用于开启Qt的事件循环。在Qt收到底层窗口管理系统发送过来的消息后，会在事件循环中将这些消息封装成Qt事件。除了来自系统的消息外，Qt事件也可以由程序内部产生，如一个定时器事件。Qt的事件是一个`QEvent`类型的对象，这些对象保存了事件的具体信息，Qt的事件对象类型和事件类型是不一样的，

- [事件对象类型](http://doc.qt.io/qt-5/events.html)：由于Qt中事件是一个C++语法概念上的对象，所有的Qt事件都是`QEvent`的子类对象。如对于一个鼠标事件，这个事件对象类型是`QMouseEvent`
- [事件类型](http://doc.qt.io/qt-5/qevent.html#Type-enum)：`QEvent`及其子类有一个`type`方法，用于获取事件类型。事件类型是一个`QEvent::Type`枚举类型，如对于一个鼠标事件，事件类型可能是`MouseButtonDblClick`,`MouseButtonPress`,`MouseButtonRelease`等。

也就是说事件对象类型可能有一到多种事件类型。这些事件接着会在事件循环中被分发到事件作用的目标组件，然后事件将会以该组件在对象树中的位置为起点，向着对象树的根节点链世传播，直到事件被接受或传播到根节点。对于一个事件，在事件处理函数中，我们通过调用`QEvent::accept()`来主动接受一个事件，表示这个事件我已经处理了，不需要再继续传播了。如果我们不想要处理这个事件，我们可以调用`QEvent::ignore()`来忽略这个事件，Qt事件循环将会将这个事件传递给当前组件的父组件，也就是对象树上的父节点去处理。如果我们两个方法都没手动调用，那么这个事件将会默认接受。
![Qt事件分发](https://blog-1251989759.cos.ap-guangzhou.myqcloud.com/blog/qt_event/event_dispatch.png)

Qt的事件可能容易和信号混为一谈，但实际上二者并无太大关系。二者的区别如下

- 信号是由Qt对象发出，是一个函数，
- 事件可能来自于外部，也可能是内部的对象生成，是一个`QEvent`对象

但事件往往是信号的诱因。如用户点击了一个`QPushButton`按钮，这个时候点击操作是一个事件，按钮在收到点击事件后会发出一个`onClicked`的信号。这就导致了在Qt中，我们很多时候使用一个控件不需要去关心事件，而只需要去关心控件的信号就可以了。和其他UI框架相比，如在`Android`中，如果我们需要关心一个按钮是否被点击，我们需要为按钮设置一个`ClickListener`并实现`onClick`方法，也就是直接对点击事件进行处理，而Qt通过信号槽进行了更高层次的封装。

如果我们需要自定义一个控件，我们就往往需要去关心各类事件了。在收到事件后，Qt的事件循环首先会去调用对应控件的`event()`函数，该函数会对根据事件类型，对事件进行二次分发，调用不同事件类型对应的事件处理函数，如对一个`QEvent::Timer`类型的`QTimerEvent`事件会调用`timerEvent()`。

举个栗子，对于下面的程序，如果我们将鼠标移到按钮`Button C`上，按下鼠标左键会发生什么？
![示例程序窗口](https://blog-1251989759.cos.ap-guangzhou.myqcloud.com/blog/qt_event/demo_window.png)

1. 首先鼠标会发出左键按下的电信号经过转换传递给操作系统，操作系统在收到这个信号后会生成鼠标按下的消息，如在`Windows`下就是一个`MSG`结构体的消息，消息ID为`WM_LBUTTONDOWN`。
2. 接着Windows系统会将这个消息传给Qt，告诉Qt程序，有个消息要发给控件`Button C`。
3. Qt程序的事件循环在收到这个消息后，会对这个事件进行解析包装，创建一个事件类型为`QEvent::MouseButtonPress`的事件`QMouseEvent`，
4. 然后事件循环将会把这个事件传递给按钮`Button C`，也就是调用按钮`Button C`的`event(QEvent *event)`方法。
5. 按钮`Button C`在收到这个事件后，会根据事件类型`QEvent::MouseButtonPress`，调用对应的具体处理函数`mousePressEvent(QMouseEvent *e)`。注意默认的`QPushButton`类此时并不会发送`onClicked`信号，而是在鼠标抬起，收到`QEvent::MouseButtonRelease`类型的事件后才会发出信号。在这个函数中事件经过某些处理后将会被接受。一旦事件被接受后，这个事件的声明周期也就完结了。

但如果鼠标是移到`Label C`并按下鼠标左键会发生什么？

首先前4步和上面类似，经过一系列处理后会调用`Label C`的`event(QEvent *event)`方法。

1. 同上
2. 同上
3. 同上
4. 调用`Label C`的`event(QEvent *event)`方法。
5. 由于QLabel默认是不处理`QMouseEvent`事件的，也就是`Label C`会忽略这个事件。这个事件会向上传递给`Label B`。
6. 同理，`Label B`也会忽略这个事件，这个事件继续向上传递。
7. 最后事件被传递到了顶层窗口，这是一个`QWidget`，由于`QWidget`默认忽略所有事件，而且这个`QWidget`已经没有父组件了，因此这个事件将被丢弃，结束其生命周期。

## 事件处理

有时候我们可能希望改变默认的事件处理逻辑。如果我们希望在一个`QLabel`被鼠标左键点击时也能和QPushButton一样发出`onClicked`信号而且不改变其他行为，该怎么做呢？首先我们可以由于Qt基础控件中的事件处理函数都是虚函数，因此我们可以继承`QLabel`并添加`onClicked`信号，然后重写`QLable::mouseReleaseEvent()`，在函数中发出`onClicked`信号。

``` cpp
void MyLabel::mouseReleaseEvent(QMouseEvent *event) {
    if(event->buttons() & Qt::LeftButton) {
        emit onClick();
    }
}
```

这样，我们就可以关联`onClicked`信号，在用户鼠标左键按下时触发槽函数。但此时鼠标抬起事件不再会向父组件传播，因为在上面这个实现中，我们没有对事件调用`QEvent::accept()`或`QEvent::ignore`，所以Qt会默认认为该事件已经被接受，不再继续传递。为了避免这个问题，保持`QLabel`的默认行为，我们可以在重写方法中手动调用父类的同名方法，将事件交由基类去处理。由于绝大部分情况下，我们继承一个已有控件并重写其事件处理函数都是为了添加一些额外的逻辑处理，并不希望影响已有控件的原有效果，因此我们往往需要去调用基类的同名事件处理函数来保证原有处理逻辑不变。如我们继承`QPushButton`并重写了其`mouseReleaseEvent(QMouseEvent *event)`时，添加了一个改变按钮文本的处理，如果不调用`QPushButton::mouseReleaseEvent(QMouseEvent *event)`，那么我们的自定义按钮将无法发出`onClicked`信号，按钮的一些内部状态也无法被切换。除非我们本来就不希望基类控件的事件处理函数有效果，否则一定要调用基类控件的同名方法。

对于部分特殊事件，Qt的事件循环会根据事件的接受与否执行不同的处理。我们来看一个例子，对于上面的程序，我们如果希望在用户点击了窗口的关闭按钮后，弹出确定关闭程序对话框，可以重写`QWidget`的`closeEvent(QCloseEvent *event)`。

``` cpp
void Widget::closeEvent(QCloseEvent *event) {
    bool exit = QMessageBox::question(this,
                                  tr("Quit"),
                                  tr("Are you sure to quit this application?"),
                                  QMessageBox::Yes | QMessageBox::No,
                                  QMessageBox::No) == QMessageBox::Yes;
    if (exit) {
        event->accept();
    } else {
        event->ignore();
    }
}
```

我们弹出了一个对话框询问是否确认退出，然后根据用户的选择决定事件的接受与否。对于一个`QCloseEvent`事件，若其在处理之后状态为已接受（即`QEvent::isAccepted()`方法返回`true`），那么Qt将会关闭当前窗口，否则不会退出。假如这里我们两个函数都没有手动调用，则默认接受，窗口还是会被关闭。

如果我们希望处理多个事件，我们还可以重写`event(QEvent *event)`函数，在事件到达具体的事件处理函数之前进行处理。我们利用这个方法重新实现下上面这个效果。

``` cpp
bool Widget::event(QEvent *event) {
    if (event->type() == QEvent::Close) {
        bool exit = QMessageBox::question(
                            this, tr("Quit"),
                            tr("Are you sure to quit this application?"),
                            QMessageBox::Yes | QMessageBox::No,
                            QMessageBox::No) == QMessageBox::Yes;
        if (exit) {
            event->accept();
        } else {
            event->ignore();
        }
        return true; // 用于告诉事件循环该事件有没有被处理。
    }
    return QWidget::event(event);
}
```

重写`event(QEvent *event)`时我们同样要注意调用基类的同名函数用于处理其他我们不感兴趣的事件，否则会导致其他事件没有任何效果。

## 事件拦截

继承并重写一个组件的事件处理函数虽然可以实现对事件的特殊处理，但这种方式往往显得繁琐，因此Qt提供了事件过滤器。事件过滤器相关的知识在豆子的《Qt学习之路2》中的[事件过滤器](https://www.devbean.net/2012/10/qt-study-road-2-event-filter/)讲解的特别清楚，这里就不再重复。

## 参考

- [Qt 学习之路 2（18）：事件](https://www.devbean.net/2012/09/qt-study-road-2-events/)
- [Qt 学习之路 2（19）：事件的接受与忽略](https://www.devbean.net/2012/09/qt-study-road-2-events-accept-reject/)
- [Qt 学习之路 2（21）：事件过滤器](https://www.devbean.net/2012/10/qt-study-road-2-event-filter/)
- [Qt 学习之路 2（22）：事件总结](https://www.devbean.net/2012/10/qt-study-road-2-event-summary/)
- [The Event System](http://doc.qt.io/qt-5/eventsandfilters.html)
- [Event Classes](http://doc.qt.io/qt-5/events.html)
