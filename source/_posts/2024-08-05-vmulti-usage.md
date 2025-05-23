---
title: vmulti虚拟输入驱动接口文档
author: 张帆
tags:
  - 驱动
date: 2024-08-05 10:16:45
---


## 简介

[vmulti](https://github.com/djpnewton/vmulti)是一个Windows下开源的输入设备模拟驱动，支持Win7及以上系统，我们可以基于该驱动实现鼠标/键盘/触摸板/触摸屏等输入外设的模拟。

<!--more-->

## 对比

和现有的通过WIN32 API（`mouse_event`/`keybd_event`/`SendInput`）发送鼠标/键盘事件的远程控制实现相比，优缺点如下：

| 实现  | vmulti  | Win32 API  |
|---|---|---|
|  优点 | 1. 不受UIPI的限制，不会存在事件失效的情况<br>2. 不受安全软件的限制<br>3. 触摸屏模拟可以实现完美的触摸体验，如单指滑动是滚动效果，而不是选中效果，以及手势效果  |  1. 可以向特定的窗口发送事件（可能用于单窗口投屏时的反向控制）<br> 2. 不需要安装驱动<br> 3. 数据和实际显示对应，不存在偏移问题<br> |
| 缺点  | 1. 需要安装驱动，可能导致用户部分软件或外设运行异常<br> 2. 无法控制单个窗口<br> 3. 无法被多个应用访问，同一时间只能有一个应用打开虚拟驱动，可通过统一的驱动访问进程进行集中处理<br> 数据以显示器信息为基准写入，在屏幕画面有旋转/黑边等情况时会导致事件偏移，需要手动进行矫正  |  1. 受UIPI限制，低权限进程无法向高权限进程窗口发送事件，导致无法控制<br> 2. 可能受安全软件拦截<br> 3. 触摸体验极差，仅有基础的点击/长按/选中操作<br> |

## 接口文档

该驱动默认支持了多个类型的外设（触摸屏，鼠标，键盘，笔，手柄等），有些外设是我们不需要的，为了减少对用户的干扰，我们需要进行裁剪。

该项目没有接口文档，仅有一个简易的test应用供参考。经过反复测试，接口文档如下：

### 打开

遍历所有HID设备，寻找满足以下条件的设备

- `usage page`为`0xff00`
- `usage`为`0x0001`
- vid/pid为0xBA1C/0x001F或0xBA2C/0x002F或0xBA3C/0x003F或0xBA4C/0x004F或0xBA5C/0x005F

写入的数据格式为HID报文，格式如下

```
0               8               16              24            31
 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    ReportID   |  ReportLength |            data...            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

其中`ReportID`固定为`REPORTID_CONTROL`，`ReportLength`为data的实际长度，data为具体的输入事件数据，不同的输入事件格式见下文。



### 触摸

触摸数据格式如下

```
0               8               16              24            31
 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    ReportID   |                                               |
+-+-+-+-+-+-+-+-+                                               +
|                                                               |
+                                                               +
|                                                               |
+                                                               +
|                                                               |
+                                                               +
|                            TOUCH[2]                           |
+               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               |  ActualCount  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

TOUCH数据格式如下：

```
0               8               16              24            31
 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Status    |   ContactID   |             XValue            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             YValue            |             Width             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             Height            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```


各字段说明如下：

- `ReportID`：固定为`REPORTID_MTOUCH`
- `Touch[2]`：触摸点数据
- `Status`: 触摸点状态，按下和移动为`MULTI_CONFIDENCE_BIT | MULTI_IN_RANGE_BIT | MULTI_TIPSWITCH_BIT`，抬起为`MULTI_CONFIDENCE_BIT`
- `ContactID`：触摸点ID，需大于0，不同的触摸点ID必须不同，触摸点抬起后ID可复用
- `XValue`：触摸点横坐标，坐标系为以屏幕左上角为原点，范围为[0, 0xffff]
- `XValue`：触摸点纵坐标，坐标系为以屏幕左上角为原点，范围为[0, 0xffff]
- `Width`: 触摸区域宽度
- `Height`: 触摸区域高度
- `ActualCount`：触摸点数量，仅第一个报文的第一个触摸点数据中的ActualCount为实际触摸点数量，其余触摸点数据中的该字段需填入0。

Windows 1809更新后，每次必须将当前所有触摸点进行报告，每个报文可以报告两个触摸点，当触摸点大于2个时，需要拆分为多个报文发送。

### 鼠标

鼠标数据如下：

```
0               8               16              24            31
 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    ReportID   |     Button    |             XValue            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             YValue            | WheelPosition |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

各字段说明如下：

- `ReportID`：固定为`REPORTID_MOUSE`
- `Button`：左键对应`MOUSE_BUTTON_1`，右键对应`MOUSE_BUTTON_2`，中键对应`MOUSE_BUTTON_3`，实际按键的按位与结果，鼠标移动时也需要设置
- `XValue`：鼠标横坐标，坐标系为以屏幕左上角为原点，范围为[0, 0xffff]
- `YValue`：鼠标纵坐标，坐标系为以屏幕左上角为原点，范围为[0, 0xffff]
- `WheelPosition`：滚轮位置，鼠标滚轮向前滚动为正值，向后滚动为负值，滚动一次对应1单位，即Win32滚轮事件中的120（Why was WHEEL_DELTA chosen to be 120 instead of a much more convenient value like 100 or even 10? - The Old New Thing (microsoft.com)）


### 键盘

键盘数据如下：

```
0               8               16              24            31
 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    ReportID   | ShiftKeyFlags |    Reserved   |               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+               +
|                          KeyCodes[6]                          |
+               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               |
+-+-+-+-+-+-+-+-+
```

各字段说明如下：

- `ReportID`：固定为`REPORTID_KEYBOARD`
- `ShiftKeyFlags`：修饰键按位与结果，各修饰键分别对应`KBD_LCONTROL_BIT`/`KBD_LSHIFT_BIT`/`KBD_LALT_BIT`/`KBD_LGUI_BIT`/`KBD_RCONTROL_BIT`/`KBD_RSHIFT_BIT`/`KBD_RALT_BIT`/`KBD_RGUI_BIT`，GUI为Windows键/Command键/Meta键
- `Reserved`：固定为0
- `KeyCodes[6]`：当前按下的键，键值同HID keycode（https://gist.github.com/MightyPork/6da26e382a7ad91b5496ee55fdc73db2#file-usb_hid_keys-h）,该驱动仅支持最多6键无冲，即若同时按下了超过6个键（不包括修饰键），仅前六个生效。
