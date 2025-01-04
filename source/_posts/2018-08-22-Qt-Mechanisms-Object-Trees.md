---
title: Qt从0到1之机制篇 - 对象树
author: 张帆
tags:
  - Qt
  - Qt从0到1
date: 2018-08-22 16:29:43
---

[本节示例代码](https://github.com/xyz1001/QtExamples/tree/master/ObjectTrees)

C/C++一直为人诟病的一点就是内存管理，C/C++提供了直接的内存操作接口给用户，这虽然效率上的优势，但也带来了内存泄漏等问题。虽然C++中引入了智能指针，但这对于管理一个GUI程序来说，一个控件中可能包含多个子控件，子控件也可能包含多个子控件，控件上可能还包含一些不可见对象，当一个控件被删除后，附在其上的子控件和不可见对象也需要一并删除，智能指针也难以实现这样的效果。因此Qt引入了对象树系统。

<!--more-->

## 认识对象树

对象树指的是Qt中树状的管理对象的所有权的一套机制，主要是为了解决对象的生命周期管理问题。其有以下特点：

- 在对象树中，每个节点都是一个QObject或其子类对象。
- 父节点拥有子节点的所有权，即父节点被删除时，子节点也会被删除。
- 子节点会继承父节点的部分效果。如父节点隐藏时，子节点也回被隐藏；为父节点设置的QSS效果也会影响到子节点。
- 对于类型为`QWidget`或`QWidget`之类的子节点，其布局坐标系是相对于父节点的。即子控件的坐标系是相对于父控件的。

## 创建对象树

``` cpp
QObject(QObject *parent = Q_NULLPTR)
```

这是`QObject`的构造函数，我们可以看到有一个参数`parent`，这个参数就是用于指定该对象的父节点。所有继承于`QObject`的类，其构造函数中都存在这个参数。特别的，对于`QWidget`及其子类等可见控件，`parent`参数的类型为`QWidget`。
`parent`参数有一个默认参数`Q_NULLPTR`，这是一个Qt定义的宏，在`nullptr`引入C++11之前Qt就提供了这个宏用于表示空指针。若一个节点的父节点为空，则其为根节点。对于类型为`QWidget`及其子类的节点，若其为一个根节点，则其以桌面为坐标系进行布局。

## 理解对象树

对象树的实现依赖`QObject`。`QObject`内部保存了其子节点的一个列表，我们可以通过`const QObjectList &QObject::children() const`方法获取其保存的子节点列表。同时，`QObject`也保存了其父节点的指针。当一个节点被删除时，其会遍历子节点列表，将子节点也一并删除，从而实现一个树状的链式删除。除了删除子节点，为了避免其父节点在删除时又重复析构自己，一个节点删除时还会将自己从其父节点的子节点列表中移除。我们看一个例子。

``` cpp
#include <QApplication>
#include <QTimer>
#include <QWidget>

int main(int argc, char *argv[]) {
    QApplication a(argc, argv);

    QTimer timer;
    QWidget w;
    timer.setParent(&w);

    w.show();

    return a.exec();
}
```

执行这个程序，退出时输出窗口会打印以下崩溃信息

> double free or corruption (out)
> 程序异常结束。

这是因为在`main`函数中，我们定义了`timer`和`w`两个局部变量，在C++中，局部变量的析构顺序是先创建后析构，因此在main函数执行结束时，也就是我们关闭窗口时，程序会先析构`w`，由于我们将`timer`的父对象设为了`w`，也就是将`timer`加入了`w`的子节点列表，在析构`w`的时候，Qt会将所有子节点，这里就是`timer`析构，所以在`w`析构完成后`timer`已经被析构了。由于`timer`本身也是一个局部变量，因此在析构完`w`之后，程序又去析构已经被析构的`timer`，所以导致重复释放，程序也就崩溃了。

这个例子反映了Qt的对象树也是存在一定问题的，为了避免这类问题，我们一般会将Qt的所有子节点都在堆上创建，而将根节点在栈上创建，这样就可以保证程序退出时可以正常析构我们的创建的对象，而且不会发生重复析构的问题。

上面这个例子如果我们将`QTimer timer;`和`QWidget w;`两句位置互换就没有问题，这是因为`timer`将会先析构，在析构时其会将自己从父节点，也就是`w`中保存的子节点列表中移除。这样在析构`w`时，`w`的析构函数就不会再去析构`timer`。

## 运用对象树

1. 对象树为我们提供了一个遍历整个程序中的`QObject`极其子类对象的途径。我们可以利用`QObject::dumpObjectTree()`和`QObject::dumpObjectInfo()`来展示一棵对象树的节点信息。
2. `QObject`还提供了对`QOject::findChildren`方法，我们可以通过这个方法寻找对象树上到符合某些条件，如指定类型，特定对象名称(`objectName`)的所有节点，然后对这些节点做一些统一处理。

## 参考

- [Object Trees & Ownership](http://doc.qt.io/qt-5/objecttrees.html)
- [Qt 学习之路 2（10）：对象模型](https://www.devbean.net/2012/09/qt-study-road-2-objects-model/)
