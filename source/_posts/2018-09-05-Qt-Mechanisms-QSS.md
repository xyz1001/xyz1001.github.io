---
title: Qt从0到1之机制篇 - Qt样式表
author: 张帆
tags:
  - Qt
  - Qt从0到1
  - Qt机制
  - QSS
abbrlink: 27725
date: 2018-09-05 09:50:31
---

[本节示例代码](https://github.com/xyz1001/QtExamples/tree/master/QtStyleSheet)

在Web前端开发中，有一个很重要的部分是`CSS`，用于修饰`Html`中的控件，如按钮，输入框等元素。`CSS`的出现分离了交互逻辑和视觉效果的开发，极大地方便了前端开发。因此这套设计理念被很多其他前端开发框架所借鉴，如Qt，Gtk，JavaFX等。在Qt中，默认的控件样式是平台相关的原始风格，很多时候难以满足视觉设计的要求，因此我们可以借助Qt中的`CSS` -- `Qt Style Sheet`（一般简称为`QSS`）来对控件进行修饰美化，定制化出我们想要的效果。

<!--more-->

## 语法

`QSS`的语法类似于`CSS2`的语法，因此建议在学习`QSS`时可以先去了解下CSS2相关的知识。一条`QSS`规则由作用目标和若干条声明组成，语法如下：

``` css
选择器[:子控件][:伪状态 ...] {
    属性: 值;
    ...
}
```

其中`[]`表示这部分是可选的，`...`表示该部分是可重复的。
下面是一个最简单的`QSS`规则示例，其中`QPushButton`是选择器。

``` css
QPushButton {
    color: red;
}
```

`QSS`的注释规则和`CSS`一致，使用`/* */`，类似C/C++中的块注释，注意不要使用`//`，这样是无效的，反而会导致解析出错，`QSS`加载异常。

## 选择器

选择器用于指定`QSS`规则的作用于哪些目标控件。Qt支持所有`CSS2`中定义的[选择器](http://www.w3.org/TR/REC-CSS2/selector.html#q1)，下面是我比较常用的一些选择器。

| 选择器类型 | 格式            |
| ---        | ---             |
| 类型选择器 | 类名            |
| ID选择器   | 类名#对象名     |
| 属性选择器 | 类名[属性="值"] |
| 全局选择器 | *               |
| 类选择器   | .类名           |

- 类型选择器
 类型选择器通过指定一个类名来确定`QSS`规则的作用目标。所有该类及其子类的对象均会收到该`QSS`规则的影响。如上面的例子中，所有类型为`QPushButton`及其子类的控件的文字颜色均会被设置为红色

- ID选择器
 ID选择器需要指定类名和控件对象名，需要注意的是这里的控件对象名不是对象的变量名，而是通过`QObject::setObjectName(const QString &name)`方法设置的对象名，多个对象可以设置同一个对象名。该选择器表示当前`QSS`规则将会作用于类型为指定类及其子类且类型名为指定名称的控件。如下面的例子中，所有类型为`QPushButton`及其子类且对象名为`ok`的控件的按钮文字颜色都会被设置为红色。

 ``` css
 QPushButton#ok {
     color: red;
 }
 ```

- 属性选择器
 属性选择器需要指定类名和属性值。属性选择器表示当前`QSS`会作用于所有类型为指定类及其子类且具有值为给定值的属性的控件。所有支持`QVariant::toString()`方法的Qt属性都可以用在这里，Qt属性可以参考[Qt属性机制](http://doc.qt.io/qt-5/properties.html)。无论设置的值的类型是什么，这里的值统一用字符串表示，如属性值类型为`bool`，那么属性选择器中等号后面应该是`"false"`或`"true"`。Qt对象除了一些自带的默认属性外，我们还可以在运行期通过`bool QObject::setProperty(const char *name, const QVariant &value)`来添加和修改属性。因此我们可以利用属性选择器实现`QSS`的动态改变。下面的例子表示所有类型为`QPushButon`及其子类且具有属性值为`1`的属性`order`的控件的文字颜色将会被设置为红色。

 ``` css
 QPushButton[order="1"] {
     color: red;
 }
 ```

- 全局选择器
 全局选择器表示当前`QSS`规则会作用于所有的Qt控件上。如下面的例子，所有的`QWidget`极其子类控件的文字都会被设置为红色。

 ``` css
 * {
     color: red;
 }
 ```

- 类选择器
 类选择器和类型选择器的区别在于类选择器在类名前加了一个`.`，表示只对该类的对象起作用，而不会影响到该类的子类对象。如下面的例子中的`QSS`规则只会将类型为`QPushButton`的控件的文字设置为红色。

 ``` css
 .QPushButton {
     color: red;
 }
 ```

这里需要注意的一点是如果类声明在命名空间内部，我们需要在类名前添加命名空间信息，语法为`[命名空间-- ...]类名`，其中多个命名空间之间需要使用`--`连接。例如对于下面的类

``` cpp
namespace out {
namespace inner {
class MyButton : public QPushButton {
    ...
}
}
}
```

要在类型选择器中使用该类`QSS`，我们可以使用以下语法

``` css
out--inner--MyButton {
    ...
}
```

## 子控件

Qt提供了众多的控件，有一些复杂控件本身又有多个组成部分。如`QSpinBox`，其可以很清楚的看到还包括了一个向上的按钮和一个向下的按钮。对于这些控件，我们经常需要对具体的某个组成部分进行样式调整。为了更方便地对这些控件进行精准的调整，我们可以在QSS中指定子控件，然后将子控件看做一个独立的控件来调整样式。不同的控件可能包括不同的子控件，我们可以查阅[Qt Style Sheets Reference](http://doc.qt.io/qt-5/stylesheet-reference.html)来查询某个控件支持哪些子控件。

如对于[`QSpinBox`](http://doc.qt.io/qt-5/stylesheet-reference.html#qspinbox-widget)，我们可以看到其除了直接包含了向上按钮`up-button`和向下按钮`down-button`，这两个子控件分别还包含了一个子控件，分别是向上箭头`up-arrow`和向下箭头`down-arrow`，也就是说，子控件中是可以继续包含其他子控件的。我们可以通过语法`选择器::子控件`来将`QSS`规则的作用目标设为子控件。

对于子控件，其支持两个特殊的属性`subcontrol-origin`和`subcontrol-position`。这两个属于用于对子控件进行定位。

- `subcontrol-origin`
 由于`CSS`使用[盒子模型](https://www.w3schools.com/css/css_boxmodel.asp)，一个盒子模型从内到外分为`content`，`padding`，`border`和`margin`。那么对于子控件，其位于其父控件的那部分呢？这就是`subcontrol-origin`属性的作用。该属性用语确定子控件位于父控件盒子模型中的哪一部分。该属性的默认值为`padding`，即子控件默认位于父控件的`padding`区域。

- `subcontrol-position`
 该属性用于设置子控件的相对于父控件的位置，值类型为[`Alignment`](http://doc.qt.io/qt-5/stylesheet-reference.html#alignment)，即对齐方式。

下面的例子中，我们将QSpinBox中的按钮移到左边，并将其宽度设为5px。

``` css
QSpinBox::up-button {
    subcontrol-position: left top;
    width: 20px;
}

QSpinBox::down-button {
    subcontrol-position: left bottom;
    width: 20px;
}
```

## 伪状态

对于同一个控件，我们经常可以看到在不同的场景下其样式也是不一样的。如对于同一个按钮，正常情况下和鼠标按下时效果是不一样的。为了实现这种效果，我们可以在`QSS`规则中指定控件的伪状态，伪状态是源自CSS中的术语（stack overflow上关于[为什么叫伪状态的讨论](https://stackoverflow.com/questions/1281407/why-is-a-pseudo-class-so-called)）。伪状态的语法为`选择器[:伪状态 ...]`，注意这里是一个冒号。伪状态描述了一个特定的场景，如被按下`pressed`，被选中`checked`等。不同的控件支持的伪状态类型不一样，我们同样可以通过[Qt Style Sheets Reference](http://doc.qt.io/qt-5/stylesheet-reference.html)来查看具体的控件支持哪些伪状态。对于同一段`QSS`，我们可以同时指定多个互不冲突的伪状态，只有当指定的伪状态同时满足时`QSS`规则才会生效。如下面的`QSS`，将只对已被选择且鼠标悬浮在其上的按钮生效。

``` css
QPushButton:hover:checked {
    background: gray;
}
```

我们可以通过在伪状态前添加`!`表示非该状态下。

## 冲突

对于一个具体的控件，可能会受到多条`QSS`规则的影响，这个时候若多条`QSS`规则中设置了同样的属性就可能会发生冲突。`QSS`的最小冲突单位是声明，而不是`QSS`规则，即两条`QSS`规则均可生效，则其中没有冲突的声明将会正常生效，而冲突的声明则会根据冲突规则决定是否生效。在发生冲突时，Qt的处理规则如下

1. 使用满足要求的
2. 使用作用范围更小的（即更具有特异性的）
3. 使用最后设置的

这三条规则的优先级依次降低。其中特异性有具体的计算规则，可以参考`CSS`中的[特异性计算规则](https://www.w3.org/TR/CSS2/cascade.html#specificity)。考虑下面的`QSS`

``` css
/* 规则1 */
QWidget{
    background-color: yellow;
}

/* 规则2 */
QPushButton {
    background-color: red;
}

/* 规则3 */
QPushButton:checked {
    background-color: green;
    color: red;
}

/* 规则4 */
QPushButton:pressed {
    background-color: blue;
}

/* 规则5 */
QPushButton:hover {
    background-color: white;
    color: green;
}

/* 规则6 */
QPushButton:pressed:hover {
    background-color: pink;
}
```

将该`QSS`设置在主窗口上，运行后我们可以看到以下现象：

1. 主窗口(`QWidget`)背景被设置为黄色，而按钮`QPushButton`背景被设为红色。这是因为因为对于主窗口，只有规则1满足要求，所以规则1生效，而对于按钮，规则1和规则2均可生效，但相对于规则1，规则2的作用范围更小，因此使用规则二。
2. 我们将鼠标悬浮在按钮上时，按钮的背景色变为白色，字体颜色变为绿色。这是因为此时规则5也满足了要求，且规则5相对与规则1,2，作用范围更小。
3. 接着我们在按钮上按下鼠标时，按钮背景又变成粉红色，而字体依然是绿色。此时规则1，2, 4，5，6均满足，其中规则6作用范围最小，因此背景颜色属性被设为粉红色。而由于规则4中还有字体颜色的声明，该声明与规则6并不冲突，因此规则4的第二条声明也是生效的。
4. 然后我们松开鼠标，此时按钮恢复成白色背景和绿色字体，此时规则1,2,3,5均满足，先排除范围较大的规则1,2，规则3,5的作用范围是同一层级的，无法区分范围大小，此时根据`QSS`冲突处理规则第三条，最后被设置的`QSS`规则将生效，即规则5将生效。
5. 最后我们将鼠标从按钮上移开，按钮变成绿色背景，红色字体，根据上一步，此时规则5已失效，因此作用范围最小的规则只有规则3，因此规则3生效。

## 继承

当我们为某个控件设置了`QSS`后，该`QSS`将会对递归作用于该控件及该控件下的所有子控件。因此对于一些全局的`QSS`设置，如字体等，我们可以直接将`QSS`设置在`qApp`上。继承同样可能带来冲突，对于继承产生的冲突解决规则是优先使用最近继承的`QSS`规则，即为该控件设置的`QSS`规则优先于父控件的`QSS`规则，父控件的`QSS`规则优先于祖父控件的`QSS`规则，以此类推。同样的，冲突的最小单位也是声明。对于上面的例子，我们为主窗口设置了`QSS`，但并未为按钮设置`QSS`，由于按钮是主窗口的子控件，因此其继承了主窗口的`QSS`，主窗口的`QSS`对其同样有效果。

## 使用

`QWidget`类提供了一个`void setStyleSheet(const QString &sheet)`方法，我们可以通过该方法来加载`QSS`。个人的习惯是将`QSS`放到单独的文件中，运行时再去读取该文件的内容并设置。由于不同的Qt控件的`QSS`差别很大，其中也有不少需要注意的问题，因此后续还会再针对具体的控件进行`QSS`使用讲解，这里就不过多展开。下面是使用`QSS`时的一些通用的注意事项和最佳实践。

- 为自定义的QWidget子类设置`QSS`时，我们需要添加以下`paintEvent`的实现

 ``` cpp
 void CustomWidget::paintEvent(QPaintEvent *) {
     QStyleOption opt;
     opt.init(this);
     QPainter p(this);
     style()->drawPrimitive(QStyle::PE_Widget, &opt, &p, this);
 }
 ```

- 使用`border`属性同时设置边框宽度，样式和颜色时，三者必须严格保持宽度，样式，颜色的顺序，否则设置无效

 ``` css
 border: 1px solid red; /* work     */
 border: solid 1px red; /* not work */
 border: red 1px solid; /* not work */
 border: red solid 1px; /* not work */
 ```

- 在调用`setStyleSheet`方法`QSS`并不立即生效。如果我们在`QSS`设置了类似于字体大小等属性，我们在加载了`QSS`后是无法立即拿到设置的属性值的。在使用需要依赖`QSS`设置的属性的方法时一定要特别注意。

## 参考

关于`QSS`的Qt官方文档还是非常详细的，尤其是`Qt Style Sheets Reference`，平时使用时需要多翻一翻。

- [Qt Style Sheets](http://doc.qt.io/qt-5/stylesheet.html)
- [The Style Sheet Syntax](http://doc.qt.io/qt-5/stylesheet-syntax.html)
- [Qt Style Sheets Reference](http://doc.qt.io/qt-5/stylesheet-reference.html)
- [Qt 之 QSS（样式表语法）](https://blog.csdn.net/liang19890820/article/details/51691212)
- [QSS总结以及最近做的Qt项目](https://www.cnblogs.com/wangqiguo/p/4960776.html#_label3)
