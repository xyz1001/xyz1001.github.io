---
title: 在库中开启Qt事件循环
author: 张帆
tags:
  - Qt
  - C++
date: 2019-05-04 11:04:16
---

[示例代码](https://github.com/xyz1001/QtExamples/tree/master/LibraryWithQt)

对于很多开发者来说，Qt仅仅是一个用于开发GUI程序的库，但实际上，Qt官方一直在致力于将Qt打造成一个跨平台的开发框架。Qt中提供了大量的基础设施和非GUI库可供我们在开发非GUI程序时所用，如网络相关的QNetwork模块，音视频相关的QMutilMedia模块，还有核心的QtCore模块。这些模块都已经相当成熟和完善，而且都具有优秀的跨平台性能。在和其他语言，如Java，python相比，库的相对不够丰富是C++的一个缺陷所在，除了标准库，boost等之外，Qt其实也是一个可以考虑的强大的补充。

如果我们只是打算使用诸如QString，QTL等基础设施，只需要链接QtCore这个库即可。然而Qt很多更为强大的特性和类，如信号槽，QTimer等，必须依赖Qt事件循环才能正常工作。如果使用我们开发的库的可执行程序恰好也是基于Qt开发，那么一切工作正常，然而，作为库的开发者，我们不能对库的使用者做出假设，因此如果我们需要使用信号槽等依赖Qt事件循环的特性时，我们就需要在库中开启Qt事件循序。

<!--more-->

## 可执行程序中的事件循环

我们先考虑在一个基于Qt的可执行程序中我们是如何开启事件循环的。相信下面的代码有接触过Qt开发的人肯定不会陌生。

``` cpp
#include "widget.h"
#include <QApplication>

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    Widget w;
    w.show();

    return a.exec();
}
```

这是通过Qt Creator创建Qt项目时`main.cpp`中的默认实现。这段代码中，我们创建了一个`QApplication`对象`a`并将启动参数传递给a，然后创建其他对象，最后通过`QApplication::exec`方法在主线程开启了一个事件循环，程序将会阻塞在`exec`这一行直到我们调用`QApplication::exit`方法退出事件循环。

## 库中的事件循环

根据上面的分析，我们可以很快想到在库中开启事件循环的方法，就是开启一个线程，然后在这个线程中创建`QCoreApplication`并开启事件循环。下面本文将以对QTimer的简单封装为例来说明。

``` cpp
// timer.h
#pragma once

#include <memory>

class QTimer;

class IListener {
public:
    virtual ~IListener() = default;

public:
    virtual void OnTimeout() = 0;
};

class Timer {
public:
    Timer(std::shared_ptr<IListener> listener);
    ~Timer();

public:
    void Start(int msec);
    void Stop();

public:
    std::shared_ptr<IListener> listener_;
    QTimer *timer_ = nullptr;
};
```

这是我们需要对外暴露的头文件，可以看到这个头文件中，除了一个前置的`QTimer`声明（当然，可以通过[PIMPL](https://zh.cppreference.com/w/cpp/language/pimpl)手法做进一步的隐藏），已经将和Qt相关的部分完全隐藏了起来。
`QTimer`是一个严重依赖Qt事件循环的工具类，因此我们需要创建一个事件循环。

``` cpp
// qeventloop_thread.h
#pragma once

#include <QCoreApplication>
#include <QThread>

class QEventLoopThread : public QThread {
public:
    // 单例
    QEventLoopThread(const QEventLoopThread &) = delete;
    QEventLoopThread operator=(const QEventLoopThread &) = delete;

    ~QEventLoopThread();

public:
    static QEventLoopThread &GetInstance();

protected:
    void run() override;

private:
    explicit QEventLoopThread(QObject *parent = nullptr);

private:
    QCoreApplication *app_{};
};
```

这里我们使用继承`QThread`并重写`QThread::run`方法的形式来开启一个线程。由于事件循环只能存在一个，因此这里使用了单例模式。`QCoreApplication`是`QApplication`的基类，这里我们不需要GUI相关的操作，不需要使用`QApplication`类。

``` cpp
// qeventloop_thread.cpp

#include "qeventloop_thread.h"

QEventLoopThread::QEventLoopThread(QObject *parent) : QThread(parent) {
    start();
}

QEventLoopThread::~QEventLoopThread() {
    app_->quit();
    // delete app_;  // 会导致崩溃，原因不明
    quit();
    wait();
}

QEventLoopThread &QEventLoopThread::GetInstance() {
    static QEventLoopThread event_loop;
    return event_loop;
}

void QEventLoopThread::run() {
    // 如果已经是Qt程序，则不再创建QCoreApplication对象
    if (app_) {
        return;
    }
    // 开启事件循环
    int argc{};
    char *argv{};
    app_ = new QCoreApplication(argc, &argv);
    app_->exec();
}
```

实现很简单，我们在`QAppThread::run`中做了类似于可执行程序中main方法的工作。首次调用`QAppThread::GetInstance`会触发`QAppThread::run`函数的执行。

然后我们去实现对外的接口。

``` cpp
// timer.cpp

#include "timer.h"

#include <QTimer>

#include "qeventloop_thread.h"

Timer::Timer(std::shared_ptr<IListener> listener) : listener_(listener) {
    timer_ = new QTimer;
}

Timer::~Timer() {
    delete timer_;
}

void Timer::Start(int msec) {
    timer_->moveToThread(&QAppThread::GetInstance()); // 设置timer_所在线程
    QObject::connect(timer_, &QTimer::timeout,
                     [this] { listener_->OnTimeout(); });
    QMetaObject::invokeMethod(timer_, "start", Qt::QueuedConnection,
                              Q_ARG(int, msec));
}

void Timer::Stop() {
    QMetaObject::invokeMethod(timer_, "stop", Qt::QueuedConnection);
}
```

对于`QObject`及其子类对象，都有一个所在线程的概念，这个线程必须是主线程或`QThread`线程，默认为创建该对象的线程。这个线程会影响到该对象的槽函数的执行线程。这里我们需要将创建的`QObject`及其子类对象移动到我们事件循环所在线程，否则该对象的槽函数不会被执行。这里还有一个需要注意的地方，`QTimer`要求必须在其对象所在线程调用`start`或`stop`，因此这里我们通过`QMetaObject::invokeMethod`来将对`QTimer::start`和`QTimer::stop`的调用放到`timer_`所在线程的事件队列中，达到在`timer_`所在线程中执行的目的。

我们写一个简单的demo可执行程序来测试一下我们的库。

``` cpp
// main.cpp
#include <iostream>

using namespace std;

#include "timer.h"

class Listener : public IListener {
public:
    void OnTimeout() override {
        static uint32_t elapsed_second = 0;
        ++elapsed_second;
        std::cout << "Elapsed time(s): " << elapsed_second << std::endl;
    }
};

int main() {
    Timer timer(std::make_shared<Listener>());
    timer.Start(1000);

    std::cin.get();
    return 0;
}
```

执行的可能输出如下

```
WARNING: QApplication was not created in the main() thread.
Elapsed time(s): 1
Elapsed time(s): 2

QObject::killTimer: Timers cannot be stopped from another thread
QObject::~QObject: Timers cannot be stopped from another thread
Press <RETURN> to close this window...
```

这里首先会有一个警告，提示说`QApplication`不是在主线程创建，由于这里我们在库里使用，必须在子线程中创建，因此这个警告可以忽略。
然后在程序退出时会提示在其他线程中调用了`QTimer::stop`方法，这时因为`Timer`对象的析构函数是在主线程中调用，间接地导致了在主线程中调用了`QTimer::stop`方法。我们需要在退出前保证`Timer`已经停止。对`Timer`的析构函数修改如下：

``` cpp
Timer::~Timer() {
    // Qt 5.10之前版本
    // QMetaObject::invokeMethod(timer_, "stop", Qt::BlockingQueuedConnection);
    // Qt 5.10及之后版本
    QMetaObject::invokeMethod(timer_, &QTimer::stop, Qt::BlockingQueuedConnection);
    delete timer_;
}
```

我们通过`Qt::BlockingQueuedConnection`标志来确保`stop`方法被调用完成后再去执行`delete`操作，否则还是可能会导致析构时`timer_`尚未停止。

## 总结

相对来说，在库中使用Qt事件循环相关的特性会麻烦一些，但和Qt提供的丰富的基础库和机制相比，这点牺牲还是值得的。如果对库的体积要求不大，是完全可以考虑在库的开发中引入Qt的。本文的只是简单介绍了如何在库中引入Qt事件循环，在代码的非侵入性，线程安全性等方面还有很大的改进空间，希望读者在使用时可以进一步完善。
