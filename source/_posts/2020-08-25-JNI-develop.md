---
title: JNI开发心得
author: 张帆
tags:
  - JNI
  - C++
date: 2020-08-25 17:11:20
---

## 前言

断断续续接触 JNI 开发也有三年多了，一直都想做些记录，但又觉得难以写出新意，网上随便一搜非常多的 JNI 入门教程。近期需要给同事做一个 JNI 开发的分享，因此通过本文做些记录。本文不是一个 JNI 入门教程，而是会侧重于 **C++** 进行 **Android** JNI 开发的一些最佳实践以及辅助工具的介绍，帮助大家更好的学习 JNI。

在使用 JNI 作为中间层将 C++ 和 JAVA 粘合起来时，最关键的是实现两个语言间类与类的交互。如果有一个 JAVA 类，该类实现了某些功能，我们需要在 C++ 中访问该类的方法，通常我们会创建一个与之对应的 C++ 适配类，然后通过该 C++ 类来调用对应的 JAVA 类的方法，反之亦然。下面我们根据适配类调用实现类的方向分成 JAVA->C++ 和 C++->JAVA 两部分来介绍。

<!--more-->

### JAVA->C++

此时实现类为 C++，适配类为 JAVA 类。在适配类中我们会添加一系列带有关键字`native`的函数。例如如下的实现类和适配类

``` cpp
// C++ 层实现类
class calculator {
public:
    void Plus(int a);
    void Minus(int a);
    void Multi(int a);
    void Divide(int a);

    int GetResult() const;

private:
    int result_ = 0;
};
```

``` java
// JAVA 层适配类
package com.example.jni;

public class Calculator {
    public native void plus(int a);
    public native void minus(int a);
    public native void multi(int a);
    public native void divide(int a);

    public native int getResult();
}
```

由于只有 C 语言提供了稳定的 ABI，因此`native`函数对应的 JNI 函数是 C 函数，函数原型十分复杂。我们可以利用 jdk 中的 javah 工具或 Android Studio 帮助我们自动创建对应的 JNI 方法。在 Android Studio 中添加 javah 外部工具的配置如下：

![Android Studio 添加 javah 外部工具](android_studio_javah.png)

对上面的适配类生成的 JNI 接口如下

``` c
#include <jni.h>

extern "C" {

JNIEXPORT void JNICALL
Java_com_example_jni_Calculator_plus(JNIEnv *env, jobject thiz, jint a) {
    // TODO: implement plus()
}

JNIEXPORT void JNICALL
Java_com_example_jni_Calculator_minus(JNIEnv *env, jobject thiz, jint a) {
    // TODO: implement minus()
}

JNIEXPORT void JNICALL
Java_com_example_jni_Calculator_multi(JNIEnv *env, jobject thiz, jint a) {
    // TODO: implement multi()
}

JNIEXPORT void JNICALL
Java_com_example_jni_Calculator_divide(JNIEnv *env, jobject thiz, jint a) {
    // TODO: implement divide()
}

JNIEXPORT jint JNICALL
Java_com_example_jni_Calculator_getResult(JNIEnv *env, jobject thiz) {
    // TODO: implement getResult()
}
}
```

我们可以发现每个 JNI 方法都有两个参数，其中`env`是 C++ 通向 JAVA 的桥梁，从名称中我们可以看出，该变量表示的是 JAVA 的环境，即上下文。在 C++ 层访问 JAVA 的接口几乎都需要通过该变量来实现。一个需要注意的是这个变量是和线程强相关的，每个线程最多只能有一个`JNIEnv`实例，一个线程里获得的`JNIEnv`变量是无法在另一个线程中使用的（实际上这个变量在 JVM 内部就是通过`thread_local`变量进行保存的）。
由于 JNI 使用 C 接口，而 C 接口会导致成员函数的上下文（可以简单理解为`this`指针）丢失，因此这里需要手动保存和回复上下文。例如在这里 JNI 接口实现时，我们缺少一个`Calculator`的对象来调用对应的 C++ 函数。这里有两种解决方法，一是通过全局或静态变量来保存一个`Calculator`对象，但这样会引入全局或静态变量，产生全局依赖，在有大量这样的对象时会非常混乱，难以管理，因此非常不推荐。

另一种方法则是将创建的 C++ 对象保存到 JAVA 对象中，然后在 JNI 方法中通过`thiz`取出对应的 C++ 对象。具体的做法有很多，个人比较习惯的是在调用 JAVA 适配器类的构造函数时调用对应的 C++ 实现类的构造函数动态分配一个 C++ 对象，然后将该 C++ 对象的指针强转为`long`类型保存在 JAVA 对象中，实现代码如下

``` java
// JAVA适配器类
public class Calculator {
    private long mIntance = 0;

    public Calculator() {
        mIntance = createNativeObject();
    }
    public native long createNativeObject(); 

    ...
}
```

``` c
// JNI接口

extern "C" {

....

JNIEXPORT jlong JNICALL
Java_com_example_jni_Calculator_createNativeObject(JNIEnv *env, jobject thiz) {
    auto calculator = new Calculator;
    return reinterpret_cast<jlong>(calculator);
}

}
```

这样处理后，我们就将创建的 C++ 对象指针保存到了对应的适配器类中，但这里存在内存泄露，我们通过`new`出来的`calculator`对象是不会被析构的，为了和 JAVA 适配器类生命周期保持同步，我们可以利用 JAVA 类的`finalize`方法来解决这个问题（但该方法在 JAVA 开发中是不推荐使用的）。具体实现如下（错误处理已简化）：

``` java
// JAVA适配器类
public class Calculator {
    ...

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        if (mInstance != 0) {
            releaseNativeObject();
            mInstance = 0;
        }
    }
    public native void releaseNativeObject();

    ...
}
```

``` c
// JNI接口

extern "C" {

....

JNIEXPORT void JNICALL
Java_com_example_jni_Calculator_releaseNativeObject(JNIEnv *env, jobject thiz) {
    // 查找对应的JAVA类
    jclass clazz = env->FindClass("com/example/jni/Calculator");
    assert(clazz);
    // 查找
    jfieldID id = env->GetFieldID(clazz, "mInstance", "J");
    assert(id);

    auto calculator = reinterpret_cast<Calculator*>(env->GetLongField(thiz, id));
    delete calculator;
}

}
```

这样处理后，在 JAVA 适配器对象被GC回收时就会出发对应的 C++ 对象的析构。接下来我们就可以去实现具体的 JNI 方法了。这里以`Java_com_example_jni_Calculator_plus` 方法为例，其他方法类似

``` c
// JNI接口实现
JNIEXPORT void JNICALL
Java_com_example_jni_Calculator_plus(JNIEnv *env, jobject thiz, jint a) {
    jclass clazz = env->FindClass("com/example/jni/Calculator");
    assert(clazz);
    jfieldID id = env->GetFieldID(clazz, "mInstance", "J");
    assert(id);
    auto calculator = reinterpret_cast<Calculator*>(env->GetLongField(thiz, id));

    calculator->Plus(int(a));
}
```

可以看到代码和`Java_com_example_jni_Calculator_releaseNativeObject`很类似，通常个人的习惯是会将`clazz`和`id`进行缓存以避免反复查询（需要注意的是这里的`clazz`是局部引用，若要缓存需要通过`NewGlobalRef`方法获取全局引用），并将获取 C++ 对象方法的代码抽成一个单独的函数。


### C++->JAVA

若我们需要在 C++ 层调用 JAVA 类中的方法，这里实际上是可以不用包装一个 C++ 适配类的，但通常为了代码更加优雅，我们会添加一个 C++ 适配类用于实现对 JAVA 实现类的访问。假设有这样一个 JAVA 类：

``` java
// JAVA实现类
package com.example.jni;

import android.os.Build;

public class SystemInfo {
    private String mOsVersion;
    private int mApiLevel = -1;

    String GetOsVersion() {
        if(mOsVersion == null) {
            mOsVersion = Build.VERSION.RELEASE;
        }
        return mOsVersion;
    }

    int GetApiLevel() {
        if(mApiLevel == -1) {
            mApiLevel = Build.VERSION.SDK_INT;
        }
        return mApiLevel;
    }
}
```

现在我们需要在 C++ 层调用这个类的方法，首先我们先创建一个对应的 C++ 适配类，

``` cpp
// C++适配类
class SystemInfo {
public:
    std::string GetOsVersion() const;
    int GetApiLevel() const;
};
```

这里又遇到了同样的问题，怎么获取到 JAVA 实现类的对象呢？解决方法也是一样，我们在 C++ 的构造函数创建对应的 JAVA 对象，并获取全局引用保存起来，最后在析构函数中删除。

``` cpp
// C++适配类
// system_info.h

#include <jni.h>

class SystemInfo {
public:
    SystemInfo();
    ~SystemInfo();

...

private:
    jobject object_{};
};

// system_info.cpp
SystemInfo::SystemInfo() {
    // env 从哪获取?
    jclass clazz = env->FindClass("com/example/jni/SystemInfo");
    assert(clazz);
    // 获取构造函数ID，函数名固定为<init>，注意返回值类型为V，即void
    jmethodID id = env->GetMethodID(clazz, "<init>", "()V");
    assert(id);
    jobject jsystem_info = env->NewObject(clazz, id);
    object_ = env->NewGlobalRef(jsystem_info);
}

SystemInfo::~SystemInfo() {
    // env 从哪获取?
    env->DeleteGlobalRef(object_);
}
```

这里又有了新的问题，这里我们自己调用 JAVA 层的方法，但这里我们没有`env`这个必不可少的对象。在 JAVA->C++ 的过程中，`env` 是 JNI 函数中直接提供了，但由于不能跨线程使用，这里我们需要手动获取当前线程的`env`。JNI 提供了`jint JavaVM::AttachCurrentThread(JNIEnv** p_env, void* thr_args)`方法来实现这一点，其中`p_env`为输出参数，但该函数又属于一个新的类`JavaVM`，现在的问题变为如何获取一个`JavaVM`的对象呢？
实际上在 JAVA 层通过`System.loadLibrary()`函数加载 native 动态库时，会查找该动态库中的`jint JNI_OnLoad(JavaVM *vm, void *reserved)`函数，如果找到则会自动调用，在程序退出时会自动调用（如果存在）所有已加载动态库中的`void JNI_OnUnload(JavaVM *vm, void *reserved)`方法。这里就有我们需要的`JavaVM`对象，该对象实际上是 JVM 虚拟机的一个控制接口，理论上你可以为每个进程创建多个 JavaVM 的实例，但是安卓只允许一个，因此我们可以安全的将该实例缓存到一个全局或静态变量中。

至此，我们终于可以着手获取`JNIEnv`了，代码如下：

``` cpp
// gloabl.h

#include <jni.h>

JavaVM *GetJVM();

// global.cpp

#include "global.h"

namespace {
JavaVM *jvm = nullptr;
}

JavaVM *GetJVM() {
    return jvm;
}

#ifdef __cplusplus
extern "C" {
#endif

jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    jvm = vm;

    // 这里的env是不需要释放的，因为当前线程是 JAVA 线程，不能从 JAVA 层脱离
    JNIEnv *env;
    jint ret = jvm->GetEnv((void **) &env, JNI_VERSION_1_6);
    assert(env);
    SystemInfo::CacheClass(env);

    return JNI_VERSION_1_6;
}

void JNI_OnUnload(JavaVM *vm, void *reserved) {
    jvm = nullptr;
}

#ifdef __cplusplus
}
#endif

// system_info.cpp
#include <memory>

#include "global.h"

SystemInfo::SystemInfo() {
    JNIEnv *env;
    GetJVM()->AttachCurrentThread(&env, nullptr);
    assert(env);
    // AttachCurrentThread得到的env需要使用DetachCurrentThread进行脱离，
    // 这里使用RAII来实现这一点
    std::shared_ptr<void> guard(nullptr, [env](void *){
        GetJVM()->DetachCurrentThread();
    });

    ...
}


SystemInfo::~SystemInfo() {
    JNIEnv *env;
    GetJVM()->AttachCurrentThread(&env, nullptr);
    assert(env);
    std::shared_ptr<void> guard(nullptr, [env](void *){
        GetJVM()->DetachCurrentThread();
    });

    env->DeleteGlobalRef(object_);
}
```

这里的`env`需要反复获取，很麻烦。上面我们提到`JNIEnv`是不能跨线程使用的，但对于同一线程，获取的`JNIEnv`实际上是可以缓存供本线程的后续执行的方法使用的。在 stackoverflow 的[这个回答中](https://stackoverflow.com/questions/30026030/what-is-the-best-way-to-save-jnienv)，采纳回答中就提到可以通过`pthread_key_create`和`pthread_setspecific`来实现对`JNIEnv`的缓存。我们这里为了简单，就不进行相关的处理了。

现在我们终于可以去实现相关的方法了。两个方法实现类似，这里以`std::string SystemInfo::GetOsVersion() const`为例：

``` cpp
std::string SystemInfo::GetOsVersion() const {
    JNIEnv *env;
    GetJVM()->AttachCurrentThread(&env, nullptr);
    assert(env);
    std::shared_ptr<void> env_guard(nullptr, [env](void *){
        GetJVM()->DetachCurrentThread();
    });

    jclass clazz = env->FindClass("com/example/jni/SystemInfo");
    assert(clazz);
    // 这里的函数签名纯手写非常麻烦，建议通过 Android Studio 自动补全
    // 同时这里的method是可以缓存的，建议进行缓存，如保存在成员变量中
    jmethodID method = env->GetMethodID(clazz, "getOsVersion", "()Ljava/lang/String;");
    assert(method);
    jstring jver = static_cast<jstring>(env->CallObjectMethod(object_, method));

    // 将jstring转换为std::string
    const char *ver = env->GetStringUTFChars(jver, 0);
    std::shared_ptr<const char> os_version_guard(ver, [&env, &jver](const char *ver) {
        env->ReleaseStringUTFChars(jver, ver);
    });

    return std::string(ver);
}
```

然而当我们运行时，却很有可能遇到崩溃，日志类似于：

```
2020-08-27 21:51:26.447 28362-28395/com.example.jni A/libc: /home/xyz1001/Demo/jni/app/src/main/cpp/system_info.cpp:23: SystemInfo::SystemInfo(): assertion "clazz" failed
2020-08-27 21:51:26.447 28362-28395/com.example.jni A/libc: Fatal signal 6 (SIGABRT), code -6 (SI_TKILL) in tid 28395 (Thread-2), pid 28362 (com.example.jni)
```

看上去是`jclass clazz = env->FindClass("com/example/jni/SystemInfo");`这一步获取`jclass`失败了，没有找到这个类，但这里我们反复检查，确实没写错。其实这里的根本原因是出在这里的`JNIEnv`上。上面我们提到，`JNIEnv`包含了 JAVA 的环境信息，其中有一个非常重要的值是 classpath，即 JAVA 类的查找路径，`findclass`会在这个值下的路径中查找指定的类。而我们通过`AttachCurrentThread`获取的`JNIEnv`实例中的 classpath 仅包含 JAVA 标准库的路径，`SystemInfo`这个类的路径是没有包含的。因此`findclass`找不到这个类。这个问题解决思路也很简单，既然这里的`JNIEnv`实例不行，那我们换一个 classpath 没问题的不就可以了么。哪里的`JNIEnv`一定可以获取到呢，那就是 JAVA 层创建的线程中的`JNIEnv`实例，这里有一个非常好的地方可以做这件事，那就是`JNI_OnLoad`函数中。首先该函数一定是在 JAVA 线程中执行的，二是在该函数执行前 C++ 类绝对不会被使用，也就是说调用到 C++ 函数时可以保证所需的`jclass`一定被缓存了。因此我们做如下修改：

``` cpp
// system_info.h
class SystemInfo {
    ...
public:
    ...
    static void CacheClass(JNIEnv *env);

    ...
};

// system_info.cpp
...

namespace {
static jclass gClazz = nullptr;
}

SystemInfo::SystemInfo() {
    ...
    jmethodID id = env->GetMethodID(gClazz, "<init>", "()V");
    ...
}

...

std::string SystemInfo::GetOsVersion() const {
    ...
    jmethodID method = env->GetMethodID(gClazz, "getOsVersion", "()Ljava/lang/String;");
    ...
}

int SystemInfo::GetApiLevel() const {
    ...
    jmethodID method = env->GetMethodID(gClazz, "getApiLevel", "()I");
    ...
}

void SystemInfo::CacheClass(JNIEnv *env) {
    if(gClazz == nullptr) {
        auto clazz = env->FindClass("com/example/jni/SystemInfo");
        gClazz = static_cast<jclass>(env->NewGlobalRef(clazz));
    }
}

// global.cpp
#include "global.h"

#include <cassert>

#include <memory>

#include "system_info.h"

...

jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    jvm = vm;

    JNIEnv *env;
    GetJVM()->AttachCurrentThread(&env, nullptr);
    assert(env);
    std::shared_ptr<void> env_guard(nullptr, [env](void *){
        GetJVM()->DetachCurrentThread();
    });
    SystemInfo::CacheClass(env);

    return JNI_VERSION_1_6;
}
```

这里缓存的`jclass`理论上来说是需要调用`JNIEnv::DeleteGlobalRef`来释放的，但由于这里的缓存生命周期直到程序退出时才结束，此时释放与否都意义不大了，因此这里就没有进行释放，但如果是缓存生命周期结束早于程序退出，那么务必记得释放，否则会导致内存泄露。我们再次运行程序，一切正常。

以上我们就完成了 JAVA 和 C++ 互相调用的逻辑。对于 C++ 访问 JAVA 类中的成员变量类似 JAVA->C++ 小节中我们访问 JAVA `Calculator`的`mInstance`变量，这里就不展示了。需要提一下的是在 JNI 层我们实际上是可以忽略 JAVA 层的访问控制而直接访问 JAVA 类的私有方法和私有变量的，但一般不推荐这么做，会破换上层的封装。

### 小结

在了解完上面两个部分后，相信大家对 JNI 开发有了基本的了解（[完整示例代码](https://github.com/xyz1001/BlogExamples/tree/master/jni)）。这里面比较关键的是准确识别实现类和适配类，在一些复杂的情况下我们很容易混淆。这里补充几个常见问题的处理方式：

- JAVA 层调用 C++ 函数，有一个类型为纯虚基类指针的参数。这个在回调中非常常见，库提供一个纯虚基类A，应用实现这个类并将子类传入库中。这里我们通常会在 JAVA 层提供一个具有相同接口的接口类B，然后在 JNI 层添加一个类C，继承并实现A的虚方法，具体实现为调用B的对应方法。这里的B是实现类（实际上的具体实现由其子类提供），而C是适配类。类图如下：

{% plantuml %}
    package C++ {
    interface ICppListener {
        + void Call()
    }

    Class CppClass {
        + void SetListener(std::shared_ptr<ICppListener>)
    }

    class CppListenerJni {
        + void Call()
        - jlistener_: jobject
    }

    }

    CppClass -> ICppListener
    ICppListener <|.. CppListenerJni

    package JAVA {
    interface IJavaListener {
        + void call()
    }

    class JavaListener {
        + void call()
    }

    class JavaClass {
        - mInstance: long
    }
    }

    JavaListener --|> IJavaListener
    JavaClass ..> JavaListener
    JavaClass -> CppClass
    IJavaListener <- CppListenerJni
{% endplantuml %}

- C++ 层调用 JAVA 函数，需要传递一个类型为`std::function`参数。 JAVA 中并没有与之对应的类型，这里的一个解决方法将`std::function`包装到一个适配类中，然后在 JAVA 层的实现类中通过方法调用适配类中的函数。类图如下

{% plantuml %}
    package C++ {
    Class ICppListener {
        + void CallLater(std::function callback)
    }

    class CppListenerJni {
        + void CallLater(std::function callback)
        - jlistener: jobject
    }

    class CppCallback{
        + CppCallback(std::function callback)
        + void Call()
        - callback: std::function
    }

    }

    ICppListener <|.. CppListenerJni
    CppListenerJni ..> CppCallback

    package JAVA {
    interface IJavaListener {
        + void callLater(JavaCallback callback)
    }

    class JavaListener {
        + void callLater(JavaCallback callback)
        - callback: JavaCallback
    }

    class JavaCallback {
        + void call()
        - mCppCallback: long
    }
    }

    IJavaListener <|.. JavaListener
    CppListenerJni -> IJavaListener
    JavaListener --> JavaCallback
    JavaCallback --> CppCallback
{% endplantuml %}

了解完裸写 JNI 之后，我们会发现写起来还是比较繁琐的，尤其 C++->JAVA 这部分。因此，在 JNI 的基础上，一些大牛提供了更好用的 JNI 封装库或工具，下面主要介绍两个：[JMI](https://github.com/wang-bin/JMI.git) 和 [Djinni](https://github.com/dropbox/djinni.git)

## JMI

JMI 属于 JNI 封装库，主要用于简化 C++->JAVA 这部分的使用。该库是国内开发者进行开发，提供了相对完善的中文文档和示例，代码也很简单，就一个头文件和源文件，上手还算比较简单。关于该库的具体使用这里就不展开介绍，有兴趣的可以自行阅读其中文文档和示例。对上面的示例代码使用 JMI 进行重构后([完整示例代码](https://github.com/xyz1001/BlogExamples/tree/master/jmi))，部分代码如下：

``` cpp
// global.cpp

#ifdef __cplusplus
extern "C" {
#endif

jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    jmi::javaVM(vm, JNI_VERSION_1_6);
    static_cast<jclass>(SystemInfo());
    return JNI_VERSION_1_4;
}

#ifdef __cplusplus
}
#endif


// system_info.h
#include "jmi.h"

class SystemInfo: public jmi::JObject<SystemInfo> {
public:
    using jmi::JObject<SystemInfo>::JObject;

    static const char *name() {
        return "com/example/jmi/SystemInfo";
    }

public:
    std::string GetOsVersion() const;
    int GetApiLevel() const;
};


// system_info.cpp
#define TAG(JavaFunc)             \
    struct JavaFunc : jmi::MethodTag {  \
        static const char *name() { \
            return #JavaFunc;           \
        }                           \
    }

std::string SystemInfo::GetOsVersion() const {
    TAG(getOsVersion);
    return call<std::string, getOsVersion>();
}

int SystemInfo::GetApiLevel() const {
    TAG(getApiLevel);
    auto jlevel = call<jint, getApiLevel>();
    return static_cast<int>(jlevel);
}
```

可以很明显的看到代码精简了很多。

这里提几个我在使用过程中的踩的坑。

1. 类型映射。在通过 JMI 调用 JAVA 方法时，我们在模板中指定参数或返回值类型时，如果是基本类型，则必须为 JNI 类型，即 **j** 开头的类型，如上面的`call<jint, getApiLevel>()`，即使是可以隐式转换的基本类型也需要显示转换一下，这里就不能写成`int`，而对象类型必须是 C++ 类型，如这里的`call<std::string, getOsVersion>()`，我们不能使用`jstring`。这是因为因为 JMI 中使用了 SFINAE，且只针对基本类型的 JNI 类型和对象类型的 C++ 类型进行了实例化，其他类型会导致模板实例化失败。编译报错信息类似

 ```
 undefined reference to `_jstring* jmi::detail::call_method<_jstring*, true, true>(_JNIEnv*, _jobject*, _jmethodID*, jvalue*)'
 clang++: error: linker command failed with exit code 1 (use -v to see invocation)
 ```

2. `jclass`缓存。和上面的原因一致，但由于 JMI 高度的封装，我们很容易忘记这一点，我们依然需要在`JNI_OnLoad`函数中触发 JMI 对`jclass`的缓存。这里可以通过`static_cast<jclass>(<C++适配类名>)`来实现。


## Djinni

Djinni 是由 Dropbox 开发的一个为 C++ 生成跨语言接口的工具，可以根据提供的 IDL 文件自动生成 C++ 与 JAVA 或 Objective-C 交互的代码。由于 Dropbox 已经[放弃使用C++进行跨平台开发](https://dropbox.tech/mobile/the-not-so-hidden-cost-of-sharing-code-between-ios-and-android)，全面转向原生开发，因此该项目现在也已经不再维护，该工具目前（2020/08/28）已彻底无法安装依赖，无法使用了非常可惜。但用户制作了 [docker 镜像 banuba/djinni-build-environment](https://hub.docker.com/r/banuba/djinni-build-environment)，我们可以通过docker继续使用该工具。

### IDL

Djinni 使用了一套自己定义的领域开发语言，语法还是比较简单的，对于 JNI 开发来说，核心的关键字就`enum`，`record`和`interface`三个。

| 关键字    | 作用                                                |
| ---       | ---                                                 |
| enum      | 定义枚举类型                                        |
| record    | 定义数据类                                          |
| interface | 定义接口类，通过后面的+c/+j确定实现类和适配类的位置 |

### 使用

Djinni 使用起来还是比较简单的

1. 克隆代码仓库
2. 编写 IDL
3. 调用项目代码中的`src/run`脚本自动生成对应的代码。该脚本实际最终调用的是一个 Scala 程序，如果是第一次运行，该脚本会自动安装依赖并编译，时间会比较久。
4. 实现相应的接口

Djinni 的 Demo 可以参考仓库中的 example 目录。
