---
title: Windows下文件路径相关字符编码问题
author: 张帆
tags:
  - windows
  - 字符编码
abbrlink: 2891
date: 2024-08-07 09:45:23
---

## 起源
内部某应用收到反馈，在某用户电脑上初始化日志库时会崩溃，堆栈类似如下（内部模拟，非实际堆栈）

<!--more-->

![崩溃堆栈](crash.png)

用户系统为中文环境，但用户名为韩文대한민국

从图中堆栈中可以看到，路径中存在无法解码的字符，实际上就是用户名对应的韩文。


## 分析

由于内部项目几乎都是跨平台项目，因此为了保证移植性，现有代码在Windows下对路径的处理规则如下

1. 内部代码字符串统一使用UTF8编码
2. 系统接口统一使用宽字符版本(UNICODE，对应UTF-16编码)。对于返回的宽字符，调用`WideCharToMultibyte`转换成UTF8格式的字符串；需要传入的字符串，调用`MultiByteToWideChar`从UTF8转换成宽字符
3. 标准库和文件系统相关接口（如`std::ofstream`），未明确要求，习惯上使用窄字符串版本（`std::string`/`const char*`)，编码格式通常对应ANSI。为了保证第一点，我们需要输出和输出字符串参数进行转换。
    - 对于这些接口的窄字符串输出参数，如返回值等，需要调用`MultiByteToWideChar`从ANSI转换成UNICODE格式，再调用`WideCharToMultibyte`从UNICODE转换成UTF8格式的字符串；
    - 对于这些接口的窄字符串输入参数，调用`MultiByteToWideChar`从UTF8转换成UNICODE格式，再调用`WideCharToMultibyte`从UNICODE转换成ANSI格式转换成ANSI编码字符串。
4. 第三方库接口，根据接口文档要求进行处理。

对于日志库初始化而言，其会在初始化时在用户目录创建一个日志文件，和文件系统相关的过程如下：
1. 通过`getenv`获取用户目录路径。该方法获取到的是ANSI编码格式字符串，因此进行一次规则3的转换，将ANSI转换成UTF8格式字符串
2. 将上一步得到的路径拼接上进程名和日志文件名，得到日志文件完整路径。
3. 调用spdlog的文件日志接口创建接口，传入日志文件路径。这里我们使用的是spdlog的窄字符串接口，阅读其源码，最终调用的也是系统的窄字符串文件创建接口，因此这里需要将上一步的路径字符串进行一次规则3的转换，将UTF8转换为ANSI格式字符串。

理论上上述规则是可以很好的保证字符串被合理转换了。但在崩溃堆栈中，我们可以看到，将UTF8字符串转成ANSI格式传给spdlog后，字符串内容出现问号????，对应的刚好是韩文字符的用户名，说明字符串转换出错。单步调试后发现，在UTF8->UNICODE这一步是正常的，但UNICODE→ANSI步骤时，`WideCharToMultibyte`的输出字符串就出现问号????了。这说明ANSI对应的字符编码，中文环境下也就是GBK，无法正确编码韩文字符。

结合`WideCharToMultibyte`的文档，其参数中最后两个参数，分别用于指定当遇到无法转换的字符时使用的替代字符和是否使用替代字符标志，也就是说，某些字符在使用特定字符编码时可能是无法表示的，本例中的韩文字符就是无法在ANSI（对应GBK）中表示的，因此转换实际上出现错误，但由于`WideCharToMultibyte`最后两个参数通常传NULL导致错误被忽略。


## 处理

找到问题根因，处理方案很简单，

1. 获取用户目录时，使用_wgetenv代替`getenv`，避免获取到ANSI编码的路径（路径中包含无法编码字符时，`getenv`返回的字符串就已经出错了）
2. 开启spdlog对windows宽字符路径支持，并在调用对应接口时，将UTF8字符串转换成UNICODE格式传入
3. `WideCharToMultibyte`方法调用后添加最后一个参数的检查逻辑

## 反思
虽然对本问题的解决方法很简单，但该问题反应了一个很严重的问题，即Windows下所有标准库/第三方库/系统库中和文件系统相关且以ANSI字符串做出参/入参的方法都是不安全的。例如std::fstream,std::filesystem:path等，只要路径中包含当前字符编码无法表示的字符就会导致异常。因此，我们应该禁止使用这些API，而是使用对应的宽字符版本代替，如std::wfstream等。对于C++标准库，应该禁用以下方法：

- `std::ofstream`/`std::ifstream`以`const char*`/`std::string`为入参的构造方法
- `std::ofstream`/`std::ifstream`以`const char*`/`std::string`为入参的`open()`方法
- `std::filesystem::path`以`const char*`/`std::string`为入参的构造方法
- `std::filesystem::path`以`const char*`/`std::string`为入参的`operator/()`和`append()`方法
- `std::filesystem::path::to_string()`方法

推荐的做法是
1. C++17到C++20之前版本，Windows下调用以`wstring`/`const wchar_t*`为入参的对应方法构造`std::filesystem::path`；
2. C++17到C++20之前版本，其他平台继续使用以`const char*`/`std::string`为入参的方法；
3. C++20开始，std::filesystem::path使用以std::u8string为入参的对应方法；
4. 所有平台以`std::filesystem::path`作为参数构造`std::fstream`;

但这样又会导致跨平台代码难以编写，一个比较好的方案是参考Win32中的`tchar`实现，针对不同平台通过宏来选择不同的实现，并实现对应的工具函数。

