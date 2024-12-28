---
title: 基于VS Natvis实现FourCC值可视化
author: 张帆
tags:
  - Windows
  - video
  - IDE
date: 2024-12-28 16:21:31
---

## 背景

在涉及到音视频的代码中，我们通常会使用[FourCC格式](https://fourcc.org/)描述一个音视频格式，包括容器格式，编码格式，像素格式等，其可能的定义和部分相关方法如下

``` cpp
enum FourCC : uint32_t;

template <char A, char B, char C, char D>
[[nodiscard]] constexpr FourCC MakeFourCC() {
    static_assert((A >= 'A' && A <= 'Z') || (A >= '0' && A <= '9'), "[A-Z0-9]");
    static_assert((B >= 'A' && B <= 'Z') || (B >= '0' && B <= '9'), "[A-Z0-9]");
    static_assert((C >= 'A' && C <= 'Z') || (C >= '0' && C <= '9'), "[A-Z0-9]");
    static_assert((D >= 'A' && D <= 'Z') || (D >= '0' && D <= '9'), "[A-Z0-9]");
    return static_cast<FourCC>((static_cast<uint32_t>(A) << 0) |
                               (static_cast<uint32_t>(B) << 8) |
                               (static_cast<uint32_t>(C) << 16) |
                               (static_cast<uint32_t>(D) << 24));
}
enum FourCC : uint32_t {
    kNullFourCC = 0,

    // Video Codec
    kH264 = MakeFourCC<'H', '2', '6', '4'>(),
    kH265 = MakeFourCC<'H', '2', '6', '5'>(),

    // 省略...
};

using PixelFormat = FourCC;
using ChromaFormat = FourCC;

[[nodiscard]] inline std::string FourCCName(FourCC fourcc) {
    std::string name;
    name.push_back(static_cast<char>(fourcc & 0xFF));
    name.push_back(static_cast<char>((fourcc >> 8) & 0xFF));
    name.push_back(static_cast<char>((fourcc >> 16) & 0xFF));
    name.push_back(static_cast<char>((fourcc >> 24) & 0xFF));
    return name;
}
```

也就是说，`FourCC`事实上是一个`uint32_t`的类型，这就导致在调试时，调试器中显示的变量值是一个整型数字，很难直观看到具体内容，除非将其以十六进制显示后，再依次转换成四个字母或数字。以上文中图片为例，若类型为`FourCC`的变量`codec`的值为`875967048`，转换成16进制为`0x34363248`，再每两个字符转换成一个字母或数字，可得到`462H`，即H264的逆序（受大小端影响导致）。

![默认效果](before_import_natvis.png)

<!--more-->

## 方案调研

为了方便调试，调研发现Visual Studio提供了名为[natvis](https://learn.microsoft.com/en-us/visualstudio/debugger/create-custom-views-of-native-objects)的机制，可以自定义类型显示方式。

natvis是一个基于xml的类型描述文件，我们可以按照VS的规范编写natvis文件。基本模板如下

``` xml
<Type Name="std::vector&lt;*&gt;">
    <DisplayString>{{ size={_Mylast - _Myfirst} }}</DisplayString>
    <Expand>
        <Item Name="[size]" ExcludeView="simple">_Mylast - _Myfirst</Item>
        <Item Name="[capacity]" ExcludeView="simple">_Myend - _Myfirst</Item>
        <ArrayItems>
            <Size>_Mylast - _Myfirst</Size>
            <ValuePointer>_Myfirst</ValuePointer>
        </ArrayItems>
    </Expand>
</Type>
```

其中，`Type`中即为需要自定义显示方式的完整类型名，包含命名空间。由于是xml文件，因此特殊字符，如`<` `>` `&`等需要进行转义，图中的类型实际为`std::vector<*>`，`*`用来表示模板参数。`DisplayString`即为VS显示变量时的字符串生成规则，用两个大括号包围，内部我们可以使用一对大括号包围C++代码用于将运行结果返回。也就是说这里是可以调用代码中的函数的，但一般不建议使用，可能存在副作用，下文中就遇到了相关的问题。VS对于有函数调用的情况，默认不会显示，需要手动点击才会执行。`Expand`是我们点击变量的展开按钮后显示的列表，通常是成员变量，也可以自定义内容，如上图中，我们显示了`size`和`capability`两个值，这两个值都是计算得出。

## 确定类型

回到我们的场景。我们希望以四个字符的格式去显示`FourCC`类型，但由于我们的`FourCC`实际上是`uint32_t`类型，而natvis文档中最后一行表明其不支持对基础类型的额外定制。

> Natvis customizations work with classes and structs, but not typedefs.
>
> Natvis does not support visualizers for primitive types (for example, int, bool) or for pointers to primitive types. In this scenario, one option is to use the format specifier appropriate to your use case. For example, if you use double* mydoublearray in your code, then you can use an array format specifier in the debugger's Watch window, such as the expression mydoublearray, [100], which shows the first 100 elements.

因此我们只能换个思路。考虑到实际使用时`FourCC`类型大部分情况是放在`VideoFormat`和`AudioFormat`两个结构体中使用，因此我们可以先为这两个结构体类型添加自定义显示。

## 类型转换

### 方案一

接下来就是如何将`uint32_t`类型转换成四个字符显示。首先很自然我们会想到，既然natvis支持通过大括号来执行C++代码，我们可以直接调用`FourCCName`函数来获取`FourCC`对应的字符串，代码如下。

``` xml
<Type Name="VideoFormat">
    <DisplayString>
        {{ codec={FourccName(codec)},chroma={FourCCName(chroma)},width={width},height={height},frame_rate={frame_rate} }}
    </DisplayString>
    <Expand>
        <Item Name="Codea">FourCCName(codec)</Item>
        <Item Name="Chroma Format">FourCCName(chroma)</Item>
        <Item Name="Width">width</Item>
        <Item Name="Height">height</Item>
        <Item Name="Frame Rate">frame_rate</Item>
    </Expand>
</Type>
```

导入后发现，其存在以下两个问题

- 默认不会计算表达式值，因为可能存在副作用
- 多次计算时会出现覆盖情况

![方案一效果](method_1_effect_1.png)

如上图，默认不会显示表达式计算结果，手动点击右侧刷新后，显示如下

![方案一缺陷](method_1_effect_2.png)

当点击第一行的刷新后，Codec和Chroma的值都会显示成Y420（实际是Chroma的值）。这里推测Natvis内部仅有一个缓冲区，函数返回结果会覆盖掉缓冲区导致。考虑到这个bug可能会产生误导，带来负面影响，因此该方案行不通。

### 方案二

更好的思路是不调用函数，直接让VS把`uint32_t`视为四个字节长度字符串来显示，但这里遇到了两个问题

- 由于natvis官方文档只提到了如何将指针以字符串显示，而我们这里是拿不到成员变量地址的
- 官方文档没有说明如何指定字符串长度

这两个问题久久没能找到方案。网络上关于这块的资料也非常少。反复阅读官方文档后，发现在大括号中，官方示例中有对变量取下标的写法，是否说明这里实际上语法类似C++语法，我们可以直接对变量取地址？经过试验后发现果然可以。而字符串长度则参考了windbg中显示变量类型的语法。经过调试，最终的完整代码如下：

``` xml
<?xml version="1.0" encoding="utf-8"?>
<AutoVisualizer xmlns="http://schemas.microsoft.com/vstudio/debugger/natvis/2010">
    <Type Name="VideoFormat">
        <DisplayString>
          {{container={&amp;container,[4]s}, codec={&amp;codec,[4]s}, chroma={&amp;chroma,[4]s}, width={width}, height={height}, frame_rate={frame_rate} }}
        </DisplayString>
        <Expand>
       <Item Name="Container">&amp;container,[4]s</Item>
          <Item Name="Codec">&amp;codec,[4]s</Item>
          <Item Name="Chroma Format">&amp;chroma,[4]s</Item>
          <Item Name="Width">width</Item>
          <Item Name="Height">height</Item>
          <Item Name="Frame Rate">frame_rate</Item>
        </Expand>
    </Type>

    <Type Name="AudioFormat">
        <DisplayString>
          {{container={&amp;container,[4]s}, codec={&amp;codec,[4]s}, channels={channels}, sample rate={sample_rate}, samples per packet={samples_per_packet} }}
        </DisplayString>
        <Expand>
          <Item Name="Container">&amp;container,[4]s</Item>
          <Item Name="Codec">&amp;codec,[4]s</Item>
          <Item Name="Channels">channels</Item>
          <Item Name="Sample rate">sample_rate</Item>
          <Item Name="Samples per packet">samples_per_packet</Item>
        </Expand>
    </Type>
</AutoVisualizer>
```

其中`&`需要进行转义，`[4]`表示字符串长度为4，`s`表示以ASCII字符串格式解析。

## 导入

将该文件保存为以`.natvis`结尾的文件并放到`%USERPROFILE%\Documents\Visual Studio 2022\Visualizers`下，重启Visual Studio 2022即可。

![导入效果](after_import_natvis.png)

## 调试说明

在整个开发过程中，调试也是比较麻烦的，这里我们可以开启参考官方文档打开natvis调试信息输出

> When the debugger encounters errors in a visualization entry, it ignores them. It either displays the type in its raw form, or picks another suitable visualization. You can use Natvis diagnostics to understand why the debugger ignored a visualization entry, and to see underlying syntax and parse errors.
> 
> To turn on Natvis diagnostics:
> 
> Under Tools > Options (or Debug > Options) > Debugging > Output Window, set Natvis diagnostic messages (C++ only) to Error, Warning, or Verbose, and then select OK.
> The errors appear in the Output window.

另外，每次修改后不会立即生效，必须重启调试器。
