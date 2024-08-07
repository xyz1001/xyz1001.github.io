---
title: JNI中"No implementation found for native"错误的原因总结
author: 张帆
tags:
  - jni
date: 2018-01-16 21:58:58
---

## 说明

最近由于工作需要，我需要将C++库封装打包成`AAR`给Android项目使用，因此接触了JNI技术。由于完全没有经验，我在使用过程中遇到了不少坑，耗费了不少时间。
在我遇到的JNI错误中，其中最常遇到的一个报错便是在运行时提示"No implementation found for native"错误，错误信息类似于

> No implementation found for void com.xxx.xxx.xxx.xxx(tried Java_com_xxx_xxx_xxx_xxx and Java_com_xxx_xxx_xxx_xxx_xxx)

导致这个错误的原因有很多，我也踩了其中不少的坑。为了避免以后再犯同样的错误，通过这篇文章记录总结一下我遇到的部分原因。

注：为了简洁，本文使用了部分可能不太合适的术语，具体如下

| 术语       | 说明                                   |
| ---        | ---                                    |
| native方法 | Java文件中声明的native方法             |
| native库   | 包含native方法实现的C/C++库            |
| JNI声明    | native方法对应的C方法的声明            |
| JNI实现    | native方法对应的C方法的实现            |
| JNI头文件  | 包含native方法对应的C方法声明的C头文件 |
| JNI源文件  | 包含native方法对应的C方法实现的C源文件 |

<!--more-->

## 原因

### 未实现native方法

第一种原因也是最容易想到的，就是提示出错的方法确实没有在C/C++代码中声明或实现，只需要native方法和JNI声明以及实现一一比对，很容易找出问题。

为了有效避免这个原因，我的建议有：

- 应该尽可能使用javah来生成JNI头文件，可以通过配置Android Studio中的`External Tools`来方便使用
- 确保每一个JNI方法均得到了实现
- 在修改了native方法后，第一时间修改对应的JNI声明和JNI实现

### JNI声明不对应

由于JNI声明极其复杂，若是手动添加或修改JNI声明，稍有不慎便可能导致native方法和JNI声明不匹配。由于出错后会在括号中提示`tried Java_com_xxx`，可以对照进行比对，同时还要重点检查返回值和参数是否对应。在Android Studio中，可以通过`Ctrl + 鼠标左键`点击native方法跳转的方式检查是否对应。若存在对应的JNI声明和JNI实现，应该会有两处可跳转位置

### JNI声明未使用`extern "C"`包含

由于C++的函数存在`name mangling`，而JNI在加载方法时需要通过方法名来加载对应的方法，因此每一个JNI声明均需要包含在`extern "C"`的作用范围内，否则JNI无法查找到正确的方法地址

### JNI源文件中未包含JNI头文件

这是我遇到的第一个导致此问题的原因。如果在JNI源文件中未`include`对应的JNI头文件，则JNI源文件中的JNI实现会被当做普通的C++方法，而JNI头文件中的声明则被认为是未实现的方法

### Java代码中未加载native库

由于Android使用动态加载库的方式在运行期打开、读取native库，而不是像C/C++在编译期直接链接native库。因此如果在使用native方法时尚未对native库进行加载，则同样会导致方法未实现的报错。未加载native库的原因可能有

- 未调用`system.LoadLibrary("xxx")`或相关的加载动态库的方法
- 动态库的名称有误，特别注意的是如果使用的是`system.LoadLibrary`方法，编译出来的native库文件名称需要是`lib + 库名 + 后缀`，其中方法的参数只需要传入库名即可。
- 在调用native方法是，尚未执行加载动态库的方法。如项目中有`A.java`和`B.java`两个Java文件，其中均含有native方法，但仅在`A.java`中对native库进行了加载，那么在尚未使用`A.java`时，native库是不会加载的，此时如果先使用了`B.java`中的native方法，同样会发生方法未实现的错误。为了避免这种情况，由于JNI在加载native库的时候会检查要加载的native库是否已经加载，因此推荐在所有可能被用户直接使用的包含native方法的Java类中，均添加如下代码，这样可以保证在使用该类之前native库已被加载

``` java
static {
    System.loadLibrary("native-lib");
}
```

## 总结

其实仔细想想，造成这个错误的原因大多是由于一些细节问题导致的，但由于原因比较多，而且JNI代码本身显得有些杂乱，因而特别容易出错，而且往往需要花一定时间去找到问题。本文仅仅列举了我遇到的几种可能造成该问题的原因，无法概括全部的原因。但最诡异的问题往往是由最简单的错误造成的。因此保持细心，理清思路，便能尽可能避免这些错误的发生。

## 参考

- [深入理解 System.loadLibrary](https://pqpo.me/2017/05/31/system-loadlibrary/)
- [Android JNI 使用总结](http://blog.guorongfei.com/2017/01/24/android-jni-tips-md/)
- [JNI Tips](https://developer.android.com/training/articles/perf-jni.html)
