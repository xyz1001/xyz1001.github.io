---
title: 踩坑记---命名空间污染
author: 张帆
tags:
  - 踩坑记
  - 命名空间
abbrlink: 61027
date: 2016-10-06 17:15:53
---

## 踩坑

今天运行如下代码时:

``` c++
#include <cctype>
#include <algorithm>
using namespace std;
int main()
{
    /*other code*/
    istream_iterator in_iter(fin), eof;
    while(in_iter != eof)
    {
        string s = *in_iter++;
        string word;
        remove_copy_if(s.begin(), s.end(),
        back_inserter(word), ispunct);
    }
    /*other code*/
}
```

在mingw492_32下编译提示如下错误:

> no matching function for call to 'remove_copy_if(std::basic_string<char>::iterator, std::basic_string<char>::iterator, std::back_insert_iterator<std::basic_string<char> >, < unresolved overloaded function type>)' std::back_inserter(word), ispunct);

<!--more-->

## 出坑

提示`ispunct`是无法解析的重载函数，按照平时使用的经验，`ispunct`是`cctype`库中定义的一个用来判断一个字符是否为标点符号的函数，这儿怎么回提示重载呢？后来在SO上找到了[答案](http://stackoverflow.com/questions/27971249/what-are-the-function-requirements-to-use-as-the-predicate-in-the-find-if-from-t/27971406#27971406?newreg=70d779e948ef44c389fd40dcc5d213de)。

由于C++在标准库STL中也定义了`ispunct`函数，定义于`std`命名空间，且是一个模板函数。由于程序直接通过`using namespace std`导入了`std`命名空间，程序默认使用STL库中的`ispunct`，导致编译器直接使用了未特化的模板函数，并未使用`cctype`库中的此函数，因此编译无法通过。

正如SO上所说的，为了避免此类问题出现，我们应该禁止使用using指示，即`using namespace`，而应该使用using声明。由于`std`命名空间定义了很多标识符，直接导入全部的`std`命名空间会产生严重的命名空间污染问题。在将程序修改后，编译通过。

``` c++
#include <cctype>
#include <algorithm>

using std::string;

int main()
{
    /*other code*/
    std::istream_iterator in_iter(fin), eof;
    while(in_iter != eof)
    {
        string s = *in_iter++;
        string word;
        std::remove_copy_if(s.begin(), s.end(),
        std::back_inserter(word), ispunct);
    }
    /*other code*/
}
```

## 填坑

实际上，在谷歌的[C++代码规范](https://google.github.io/styleguide/cppguide.html#Namespaces)里面有这样一条

> You may not use a using-directive to make all names from a namespace available.

由于命名空间往往是一个黑盒，我们无法直观得查看到命名空间里面定义了哪些名称，因此直接导入一个命名空间是相当危险的事，尤其是在一个多人协作的大型项目中，每个人都有可能往命名空间中添加命名。贸然使用using指示，很有可能导致各种bug，甚至是运行时bug，同时也为后期维护埋下了地雷。
