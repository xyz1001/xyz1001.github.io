---
title: Qt从0到1之机制篇 - 信号槽
author: 张帆
tags:
  - Qt
  - 信号槽
abbrlink: 19581
date: 2018-08-17 11:01:23
---

## What

Qt提供了很多机制，其中的一个核心机制就是信号槽，信号槽在Qt程序中有着广泛的使用，是Qt有别于其他框架的一个显著特点。信号槽实际上是一种对象间通信的技术，内部使用了观察者模式。

<!--more-->

## Why

Qt的信号槽主要是为了解决对象间的通信问题。在不使用信号槽时，对象间的通信一般有两种方式，一种是A对象保持B对象的实例，然后在需要的时候调用B对象的方法，另一种是将B对象的方法作为回调函数传入A对象，在合适的时间调用。但前一种方式建立了两个对象间的强耦合关系，而后一种则使用起来比较麻烦，且可能有类型安全问题。Qt的信号槽最突出的作用就是解耦了两个对象。信号所在对象无需关注槽所在对象的信息，反之亦然。同时，Qt信号槽还提供了线程安全性，可以跨线程使用。

## How

### 信号

由于Qt的信号槽机制依赖元对象系统，因此若需要在一个类中定义一个信号，该类有几点要求
- 继承于`QObject`。发出信号的类必须是`QObject`的子类。这里有个需要注意的地方是，若该类是多重继承，必须将`QObject`置于第一继承的位置，否则会无法编译，这是由于元对象系统内部有使用虚指针相关的内容。错误提示类似于
 > moc_foo.cpp:88:13: error: ‘staticMetaObject’ is not a member of ‘Bar’
 >      { &Bar::staticMetaObject, qt_meta_stringdata_Foo.data,
 >              ^~~~~~~~~~~~~~~~
 > moc_foo.cpp: In member function ‘virtual void* Foo::qt_metacast(const char*)’:
 > moc_foo.cpp:105:17: error: ‘qt_metacast’ is not a member of ‘Bar’
 >      return Bar::qt_metacast(_clname);
 >                  ^~~~~~~~~~~
 > moc_foo.cpp: In member function ‘virtual int Foo::qt_metacall(QMetaObject::Call, int, void**)’:
 > moc_foo.cpp:110:16: error: ‘qt_metacall’ is not a member of ‘Bar’
 >      _id = Bar::qt_metacall(_c, _id, _a);
 >                 ^~~~~~~~~~~
- 类声明中第一行添加单独的一行`Q_OBJECT`。`Q_OBJECT`是Qt定义的一个宏，该宏定义了一个支持元对象系统的类需要的一些函数和成员变量。注意该宏后面无分号。
- 类声明放在头文件中。由于`MOC`工具只会处理头文件，因此若类定义放在源文件中是无法进行`MOC`处理的。

在满足了上面这些要求后，我们就可以为一个类添加信号。一个信号其实就是一个函数。信号函数有以下特点
- 无返回值或返回值无效。信号函数的返回值是没有任何效果的，因此信号函数的返回值设为`void`即可。
- 仅需进行函数声明而不能去实现，因为`MOC`会帮我们添加特定实现。
- 信号定义需要位于以`signals:`为标记的区块中。`signals`不能有`public`等关键词修饰，其实际上是一个宏，在`MOC`阶段会进行处理。

定义了一个信号后，我们需要再需要的地方通过`emit`关键词发射信号。我们可以查看到`emit`的定义如下
 > \# define emit

可以看到，emit是一个被定义为空的宏，也就是说实际上`emit`是毫无作用的，发射信号的过程就是一个函数调用的过程。Qt定义`emit`更多的是一种象征意义，表示这里发出了一个信号，将信号和普通函数调用区分开来。
需要注意的是，由于`signals`默认是`public`，因此我们不仅可以在类的成员函数中发生信号，也可以在外部发生一个对象的信号。

### 槽

相对于信号，槽的要求要简单很多，`Qt4`中的槽其只能为成员函数，普通函数或静态成员函数。而`Qt5`对槽进行了一个扩充，几乎任何可调用对象都可以作为槽来使用。也就是说，成员函数，普通函数，静态函数，lambda表达式，函数对象，甚至是另一个信号，均可以作为槽来使用。但对于作为槽使用的成员函数，其有以下要求：
- 所在的类必须是`QObject`的子类且唯一继承于`QObject`或`QObject`位于第一继承位置
- 在Qt4中，我们必须将作为槽的成员函数放在`public/protected/private slots:`的区块中，因为Qt4受连接过程的限制，需要对作为槽的成员函数做一些处理。而在`Qt5`中，这一条不在严格限制，但推荐这些标记，这样会使代码意图更明确。同时我们也可以看到，槽是有访问权限控制的。一般对于无需对外暴露的槽，我们声明为`private slots:`即可。

### 连接

#### 连接方式

有了信号和槽，我们还需要将其进行一个关联。我们需要通过`QObject`的静态成员函数`connect`来实现。Qt5中，`connect`的函数原型如下：
``` cpp
QMetaObject::Connection	connect(const QObject *sender, const char *signal, const QObject *receiver, const char *method, Qt::ConnectionType type = Qt::AutoConnection)
QMetaObject::Connection	connect(const QObject *sender, const QMetaMethod &signal, const QObject *receiver, const QMetaMethod &method, Qt::ConnectionType type = Qt::AutoConnection)
QMetaObject::Connection	connect(const QObject *sender, PointerToMemberFunction signal, const QObject *receiver, PointerToMemberFunction method, Qt::ConnectionType type = Qt::AutoConnection)
QMetaObject::Connection	connect(const QObject *sender, PointerToMemberFunction signal, Functor functor)
QMetaObject::Connection	connect(const QObject *sender, PointerToMemberFunction signal, const QObject *context, Functor functor, Qt::ConnectionType type = Qt::AutoConnection)
```
其中第一种和第二种是Qt4中中，而后三种是Qt5中引入的。其中返回值主要用于判断连接是否成功，一般会忽略，再忽略参数类型和默认参数等，可以看出基本形式为
``` cpp
connect(sender, signal, receiver, method, type)
```
其中，`sender`是信号的发出者，`signal`就是信号，`receiver`是信号的接受者，`method`就是在信号发出后要执行的函数，`type`是连接类型，下面会具体讲解。接下来我们针对每个原型具体说明。
1. `QMetaObject::Connection	connect(const QObject *sender, const char *signal, const QObject *receiver, const char *method, Qt::ConnectionType type = Qt::AutoConnection)`
 该形式的信号发出者和接收着均是`QObject`类型的对象指针，而信号和槽则是字符串常量，这是因为在Qt4中是直接通过字符串来进行函数的匹配的，其使用方式类似于

 > QObject::connect(scrollBar, SIGNAL(valueChanged(int)), label,  SLOT(setNum(int)));

 其中`SIGNAL`和`SLOT`为两个宏，用于将括号中的函数转换为字符串，这种处理方式有一个严重的问题是由于使用字符串对函数进行标识，然后在运行期再进行匹配，使得编译器无法在编译期获取到信号和槽函数的类型，也就无法去判断信号和槽是否存在，参数是否匹配，而只能在运行期去判断，这往往会导致连接了错误的信号和槽，进而产生很多意想不到的bug。
2. `QMetaObject::Connection	connect(const QObject *sender, const QMetaMethod &signal, const QObject *receiver, const QMetaMethod &method, Qt::ConnectionType type = Qt::AutoConnection)`
这种形式本质上和第一种是一样的，只是通过`QMetaMethod`对函数进行了包装。该方式很少用到。
3. `QMetaObject::Connection	connect(const QObject *sender, PointerToMemberFunction signal, const QObject *receiver, PointerToMemberFunction method, Qt::ConnectionType type = Qt::AutoConnection)`
这是`Qt5`引入的新的信号槽连接方式，信号和槽的类型是`PointerToMemberFunction`，也就是成员函数指针，这样如果信号或槽函数不存在，或者是信号和槽的参数无法匹配，编译期间就会报错。
4. `QMetaObject::Connection	connect(const QObject *sender, PointerToMemberFunction signal, Functor functor)`
这种方式是用于关联信号和一个可调用对象，最常用的便是传入一个`lambda`表达式，我们也可以通过这种方式间接实现将一个非`QObject`子类的类的成员函数作为槽函数使用。
 ``` cpp
 NotQObject not_qobject;
 connect(button_, &QPushButton::clicked, [&not_qobject]{
     not_qobject.DoSomething();
 });
 ```
5. `QMetaObject::Connection	connect(const QObject *sender, PointerToMemberFunction signal, const QObject *context, Functor functor, Qt::ConnectionType type = Qt::AutoConnection)`
最后一种形式类似于第四种，但多了一个`context`参数，这个参数用于制定执行槽的事件循环所在的对象。置于事件循环等知识这里暂不具体展开。

连接时需要注意的点有
 - 信号槽的连接是多对多的，一个信号可以关联多个槽，一个槽也可以由多个信号触发，而且同一个信号和槽可以重复关联。
 - 连接信号和槽时要求信号的参数不能大于槽的参数。
 - 避免死循环，如槽函数中又触发了连接的信号。相关案例可以参考我之前的一篇文章[Qt自定义控件之SeekBar](http://xyz1001.xyz/52996.html)

#### 连接类型

在`connect`函数原型中，我们还有最后一个参数`type`未提到，该参数有一个默认参数，一般情况下使用默认参数即可。这部分内容将会在讲完Qt的事件循环后专门讲解。

### 断开连接

除非手动提前断开信号槽，否则信号槽一旦建立会一直有效，直到信号的发出者或接受者中的任意一方被析构，与被析构对象相关的所有信号槽将会被自动断开，因此我们无须担心信号槽会发生野指针相关的问题。某些情况下我们可能希望提前断开信号槽的连接，这时就需要使用到`disconnect`函数，其有普通成员函数和静态成员函数两种形式。普通成员函数原型为
``` cpp
bool disconnect(const char *signal = nullptr, const QObject *receiver = nullptr, const char *method = nullptr) const
bool disconnect(const QObject *receiver, const char *method = nullptr) const
```
静态成员函数的原型为
``` cpp
bool disconnect(const QObject *sender, const char *signal, const QObject *receiver, const char *method)
bool disconnect(const QObject *sender, const QMetaMethod &signal, const QObject *receiver, const QMetaMethod &method)
bool disconnect(const QMetaObject::Connection &connection)
bool disconnect(const QObject *sender, PointerToMemberFunction signal, const QObject *receiver, PointerToMemberFunction method)
```

其中，除了静态成员函数的第三种，我们需要传入一个连接时的返回值来断开一个特定的连接，其他方式都可以通过将调用特定参数为0或者`nullptr`来实现一个类似于通配符的作用。下面这段摘录自Qt的[官方文档](http://doc.qt.io/qt-5/qobject.html#disconnect)
1. 断开所有与对象myObject发出的信号相连接的所有信号槽:
 ``` cpp
 disconnect(myObject, 0, 0, 0);
 ```
 等同于调用普通成员函数
 ``` cpp
 myObject->disconnect();
```
2. 断开所有连接到`myObject`对象的`mySignals`信号的信号槽:
 ``` cpp
 disconnect(myObject, SIGNAL(mySignal()), 0, 0);
 ```
 等同于调用普通成员函数
 ``` cpp
 myObject->disconnect(SIGNAL(mySignal()));
 ```
3. 断开制定的一个信号槽连接:
 ``` cpp
 disconnect(myObject, 0, myReceiver, 0);
 ```
 等同于调用普通成员函数
 ``` cpp
 myObject->disconnect(myReceiver);
 ```

## Reference

- [Signals & Slots](http://doc.qt.io/qt-5/signalsandslots.html)
- [Qt 学习之路 2（4）：信号槽](https://www.devbean.net/2012/08/qt-study-road-2-signal-slot/)
