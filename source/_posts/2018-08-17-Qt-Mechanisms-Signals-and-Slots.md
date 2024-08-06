---
title: Qt从0到1之机制篇 - 信号槽
author: 张帆
tags:
  - Qt
  - 信号槽
  - Qt机制
  - Qt从0到1
abbrlink: 19581
date: 2018-08-17 11:01:23
---

[示例代码](https://github.com/xyz1001/QtExamples/tree/master/SignalsAndSlots)

## What

Qt提供了很多机制，其中的一个核心机制就是信号槽，信号槽在Qt程序中有着广泛的使用，是Qt有别于其他框架的一个显著特点。信号槽实际上是一种对象间通信的技术，内部使用了观察者模式。

## Why

Qt的信号槽主要是为了解决对象间的通信问题。在不使用信号槽时，对象间的通信一般有两种方式，一种是A对象保持B对象的实例，然后在需要的时候调用B对象的方法，另一种是将B对象的方法作为回调函数传入A对象，在合适的时间调用。但前一种方式建立了两个对象间的强耦合关系，而后一种则使用起来比较麻烦，且可能有类型安全问题。Qt的信号槽最突出的作用就是解耦了两个对象。信号所在对象无需关注槽所在对象的信息，反之亦然。同时，Qt信号槽还提供了线程安全性，可以跨线程使用。

<!--more-->

## How

### 信号

由于Qt的信号槽机制依赖元对象系统，因此若需要在一个类中定义一个信号，该类有几点要求

- 继承于`QObject`。发出信号的类必须是`QObject`的子类。这里有个需要注意的地方是，若该类是多重继承，必须将`QObject`置于第一继承的位置，否则会无法编译，这是由于元对象系统内部有使用虚指针相关的内容。错误提示类似于

 ``` cpp
 moc_foo.cpp:88:13: error: ‘staticMetaObject’ is not a member of ‘Bar’
      { &Bar::staticMetaObject, qt_meta_stringdata_Foo.data,
              ^~~~~~~~~~~~~~~~
 moc_foo.cpp: In member function ‘virtual void* Foo::qt_metacast(const char*)’:
 moc_foo.cpp:105:17: error: ‘qt_metacast’ is not a member of ‘Bar’
      return Bar::qt_metacast(_clname);
                  ^~~~~~~~~~~
 moc_foo.cpp: In member function ‘virtual int Foo::qt_metacall(QMetaObject::Call, int, void**)’:
 moc_foo.cpp:110:16: error: ‘qt_metacall’ is not a member of ‘Bar’
      _id = Bar::qt_metacall(_c, _id, _a);
                 ^~~~~~~~~~~
 ```

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

相对于信号，槽的要求要简单很多，`Qt4`中的槽其只能为成员函数，普通函数或静态成员函数。而`Qt5`对槽进行了一个扩充，几乎任何可调用对象都可以作为槽来使用。也就是说，成员函数，普通函数，静态函数，lambda表达式，函数对象，甚至是 **另一个信号**，均可以作为槽来使用。但对于作为槽使用的成员函数，其有以下要求：

- 所在的类必须是`QObject`的子类且唯一继承于`QObject`或`QObject`位于第一继承位置
- 在Qt4中，我们必须将作为槽的成员函数放在`public/protected/private slots:`的区块中，因为Qt4受连接过程的限制，需要对作为槽的成员函数做一些处理。而在`Qt5`中，这一条不在严格限制，但推荐这些标记，这样会使代码意图更明确。同时我们也可以看到，槽是有访问权限控制的。一般对于无需对外暴露的槽，我们声明为`private slots:`即可。

在信号槽内部我们可以通过`sender()`方法获取到信号的发送者，其返回值是一个`QObject *`，我们可以通过Qt提供的基于`MOC`机制的`qobject_cast`或者`std::dynamic_cast`将返回值转成我们需要的类型。这一点在实现类似与计算器的程序中尤其有用。我们可能有多个信号关联到同一个槽函数上，然后我们需要根据信号的发送者确定具体的行为。

### 连接

有了信号和槽，我们还需要将其进行一个关联。我们需要通过`QObject`的静态成员函数`connect`来实现。Qt5中，`connect`的函数原型如下：

``` cpp
// 静态成员函数
QMetaObject::Connection connect(const QObject *sender, const char *signal, const QObject *receiver, const char *method, Qt::ConnectionType type = Qt::AutoConnection)
QMetaObject::Connection connect(const QObject *sender, const QMetaMethod &signal, const QObject *receiver, const QMetaMethod &method, Qt::ConnectionType type = Qt::AutoConnection)
QMetaObject::Connection connect(const QObject *sender, PointerToMemberFunction signal, const QObject *receiver, PointerToMemberFunction method, Qt::ConnectionType type = Qt::AutoConnection)
QMetaObject::Connection connect(const QObject *sender, PointerToMemberFunction signal, Functor functor)
QMetaObject::Connection connect(const QObject *sender, PointerToMemberFunction signal, const QObject *context, Functor functor, Qt::ConnectionType type = Qt::AutoConnection)

// 普通成员函数
QMetaObject::Connection connect(const QObject *sender, const char *signal, const char *method, Qt::ConnectionType type = Qt::AutoConnection) const
```

我们可以看到，`connect`方法包括普通成员函数和静态成员函数在内，一共有六种形式，而且参数也是相当的多。我们一步步来分析这些方法。

首先我们我可看到所有形式的函数的返回值都是`QMetaObject::Connection`，我们查阅[文档](http://doc.qt.io/qt-5/qmetaobject-connection.html#details)可以看到，`QMetaObject::Connection`的作用有两个

- 用于断开一个信号槽的连接
- 检查信号槽连接是否成功

一般情况下我们会直接忽略这个返回值。

然后我们先忽略作为普通成员函数的`connect`函数，先看下作为静态成员函数的5个`connect`重载。其中前3个如果我们忽略掉参数的类型和默认参数，可以简化为同一种形式

``` cpp
QMetaObject::Connection connect(sender, signal, receiver, method, type)
```

我们先看最后一个参数`type`，表示连接类型，带有一个默认参数`Qt::AutoConnection`，一般情况下使用默认值即可。由于连接类型相关的内容涉及到其他知识点，本文将暂不做深入说明。

接下来我们看下剩下的四个参数。其中`sender`是信号的发出者，`signal`就是信号，`receiver`是信号的接受者，`method`就是槽，也就是在信号发出后要执行的函数。静态成员函数中的前三个最大的区别就是`signal`和`method`的参数类型不同。其中前两种是Qt4中就已经存在的，我们将其归为Qt4信号槽连接语法，我们先看这两种。

对于第一种，该形式的信号发出者和接收着均是`QObject`类型的对象指针，而信号和槽则是`const char *`类型，也就是一个字符串常量，这是因为在Qt4中是直接通过字符串来进行函数的匹配的，我们来看一个例子。

``` cpp
// foo.h
#ifndef FOO_H
#define FOO_H

#include <QObject>

class Foo : public QObject
{
    Q_OBJECT
public:
    explicit Foo(QObject *parent = nullptr);

signals:
    void Hello(int i);

public slots:
    void World(int i);
};

#endif // FOO_H
```

``` cpp
// foo.cpp
#include "foo.h"

#include <QDebug>

Foo::Foo(QObject *parent) : QObject(parent) {
    connect(this, SIGNAL(Hello(int i)), this, SLOT(World(int i)));
}

void Foo::World(int i) {
    qDebug() << "Hello world!"<< i;
}
```

我们可以看到我们关联信号(`Hello`)槽(`World`)时使用了`SIGNAL`和`SLOT`为两个宏，这两个宏在`Release`编译时(`Debug`时还会添加语句所在行数等信息用于调试)的定义如下

``` cpp
# define SLOT(a)     "1"#a
# define SIGNAL(a)   "2"#a
```

这里面使用了一个较少出现的宏的用法，即`#`，其功能是将其后面的宏参数进行字符串化操作，简单说就是在对它所引用的宏变量通过替换后在其左右各加上一个双引号，也就是说我们的`connect`函数经过预处理后的效果是这样的

``` cpp
connect(this, "2""Hello(int i)", this, "1""World(int i)");
```

我们忽略多出的`"1"`和`"2"`，可以看到，实际上在Qt4的信号槽连接语法中，预处理阶段我们传入的信号和槽函数会被转换成相应的字符串，我们也可以猜测到Qt会保存一个对应的字符串和函数的映射表用于运行时查找。再联想到信号和槽定义时需要放在特定的区块中，我们大致可以确定`MOC`工具在对头文件进行处理时，就是根据这些特定的区块标记(`signals`和`... slots:`)来确定哪些函数需要添加到实现信号槽所需的函数字符串与函数的映射表的。

Qt4提供的第二种信号槽连接语法实际上与第一种无异，[文档中](http://doc.qt.io/qt-5/qobject.html#connect-1)有这样一句话

> This function works in the same way as connect(const QObject *sender, const char *signal, const QObject *receiver, const char *method, Qt::ConnectionType type) but it uses QMetaMethod to specify signal and method.

因此这里不再赘述。

再回到上面这个例子，我们使用了Qt4的信号槽连接语法，但这种处理方式有一个严重的问题，如果我在编写代码时不小心将信号或槽函数打错，例如如果我们不小心将信号槽连接语句写成了如下形式

``` cpp
connect(this, SIGNAL(Helo(int i)), this, SLOT(World(int i)));
```

编译时是没有任何问题的，只有当我们运行程序时，我们才能在输出窗口中看到如下输出

> Object::connect: No such signal Foo::Helo(int i) in ../SignalsAndSlots/foo.cpp:15

如果编译的是`Release`，甚至连错误位置信息都没有。这是由于Qt4的信号槽连接语法使用字符串对函数进行标识，然后在运行期再进行匹配，使得编译器无法在编译期获取到信号和槽函数的类型，也就无法去判断信号和槽是否存在，参数是否匹配，这个缺陷导致信号槽的关联特别容易出现bug，我们只能依靠运行期间的现象和输出来排查此类bug。因此在Qt5中，Qt引入了第三种信号槽连接语法，我们将其归为Qt5信号槽连接语法。

第三个`connect`函数的信号和槽的类型是`PointerToMemberFunction`，字面意思也就是成员函数指针，如果换成新的Qt5的信号槽连接语法，上面的代码可以修改为

``` cpp
connect(this, &Foo::Hello, this, &Foo::World);
```

这里的信号和槽函数使用了成员函数指针，这样编译器就可以知道信号和槽函数的具体类型，如果信号或槽函数不存在，或者是信号和槽的参数无法匹配，编译期间就会报错。因此在实际开发中，虽然Qt5为了和Qt4兼容，依然保留了Qt的信号槽连接语法，但除非我们使用Qt4进行开发，否则我们不应再继续使用Qt4的信号槽连接语法。

任何方法都是有利有弊，Qt5虽然解决了Qt4中的信号槽连接语法的缺陷，但也随之带来了一个问题。继续以上面的例子为基础，我们在`foo.h`添加一个槽函数

``` cpp
// foo.h
void World(double d, int i);
```

然后在`foo.cpp`中添加一个空实现。这样我们其实是添加了一个槽函数`World`的重载，Qt的槽函数是允许有重载的。但是当我们再次进行编译时，编译器报错了

>  error: no matching function for call to ‘Foo::connect(Foo*, void (Foo::*)(int), Foo*, <unresolved overloaded function type>)’

编译器提示我们第四个参数是`unresolved overloaded function type`，也就是未解析的重载函数类型。我们再次将Qt4和Qt5的信号槽连接代码进行对比

``` cpp
connect(this, SIGNAL(Helo(int i)), this, SLOT(World(int i))); // Qt4
connect(this, &Foo::Hello, this, &Foo::World); // Qt5
```

我们可以看到，由于Qt5使用了成员函数指针，相对于Qt4，其丢失了参数信息，这样就导致了一旦信号函数或槽函数有重载，编译器就不知道该使用哪一个。解决这个问题有两个方法。

1. 创建指向类成员函数的函数指针，再将该指针作为参数传入。

 ``` cpp
 void (Foo::*world)(int i) = &Foo::World;
 connect(this, &Foo::Hello, this, world);
 ```

 这里我们创建了一个类型为`void (Foo::*)(int i)`的变量`world`(注意大小写)，其是一个指向类`Foo`成员函数的指针，并将`Foo`的成员函数`&Foo::World`的函数指针赋值给`world`，这里由于编译器知道`world`的类型，因此会将匹配的`Foo`的成员函数`void World(int i)`复制给`world`，然后将`world`作为参数传入，从而避免了二义性。
2. 构建一个lambda表达式进行转发。我们可以将上面的信号槽关联使用这种方式再进行修改

 ``` cpp
 connect(this, &Foo::Hello, [this](int i){
     World(i);}
 );
 ```

 这里使用到了C++11中的`lambda`表达式语法，这里不深入介绍。我们再回顾一下`connect`函数的函数原型，可以看到，将`lambda`作为槽就是静态成员函数中的第四种形式，该形式也是Qt5中新增的。这里槽实际上我们使用了一个可调用的`lambda`表达式对象，信号发出后执行的是`lambda`表达式，然后在`lambda`表达式中我们又调用了我们真正想要调用的方法，此时编译器可以正确地处理函数重载。我们通过这种间接的方式避开了问题。但这里需要注意的是这两种形式不等价，在特殊情况下会产生不同的效果，如其不支持连接类型等，这里暂不深入。另外，这种形式的槽不仅可以使用lambda表达式作为槽，任何[可调用对象](https://en.cppreference.com/w/cpp/named_req/Callable)，如仿函数，普通函数等都可以作为槽传入。

最后一种`connect`形式的静态成员函数形式其实类似与第四种类似，实际中使用较少，这里也不过多介绍。

然后我们再来看看成员函数的版本的`connect`函数，其十分类似静态成员函数中的第一个，仅仅少了一个`receiver`函数，显然是将成员函数中隐藏的`this`指针作为了`receiver`，这也就是很多文章提到说在使用静态成员函数的`connect`时，如果`receiver`为this，则可以忽略的原因。忽略掉`this`后，其实调用的就是成员函数版本的`connect`函数了。这里需要注意的的是虽然文档中成员函数版本的`connect`方法的信号和槽的类型是`const char *`，也就是看上去仅仅使用Qt4信号槽连接语法时才可以忽略掉作为`receiver`的`this`指针，但实际上使用Qt5版本的信号槽连接语法也是可以的。

信号槽连接时需要注意的点有

- 信号槽的连接是多对多的，一个信号可以关联多个槽，一个槽也可以由多个信号触发，而且同一个信号和槽可以重复关联。
- 连接信号和槽时要求信号的参数不能大于槽的参数。
- 避免死循环，如槽函数中又触发了连接的信号。相关案例可以参考我之前的一篇文章[Qt自定义控件之SeekBar](http://xyz1001.xyz/52996.html)

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

<script src="https://utteranc.es/client.js"
        repo="xyz1001/xyz1001.github.io"
        issue-term="title"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
