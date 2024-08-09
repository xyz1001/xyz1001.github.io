---
title: Win7下RegisterClassEX失败问题
author: 张帆
tags:
  - Win32
  - GUI
date: 2024-08-09 15:02:22
---

## 问题

在参考WebRTC(对应代码为`webrtc/modules/desktop_capture/win/screen_capturer_win_magnifier.cc`)实现基于放大镜接口（Magnification API）的屏幕画面采集时，遇到了一个仅在Win7下出现的bug。日志显示Win7 32位下调用`RegisterClassEX`会失败，返回错误0x87，表示参数非法。

<!--more-->

## 分析
报错函数上下文代码片段如下：

```cpp
WNDCLASSEXW wcex = {};
wcex.cbSize = sizeof(WNDCLASSEX);
wcex.lpfnWndProc = DefWindowProc;
wcex.hInstance = hInstance;
wcex.hCursor = LoadCursor(nullptr, IDC_ARROW);
wcex.lpszClassName = kMagnifierHostClass;
RegisterClassExW(&wcex);
```

根据报错，我们可以猜测是`wcex`结构体中的某个参数非法，进一步查看`WNDCLASSEX`结构体的定义，排查了其他几个参数后都没有发现问题，最后只剩下`lpfnWndProc`和`hInstance`则两个参数比较可疑。参考[WNDCLASSEXA文档](https://learn.microsoft.com/en-us/windows/win32/api/winuser/ns-winuser-wndclassexa)，`hInstance`表示`lpfnWndProc`所在模块的句柄。这里我们使用了系统提供的`DefWindowProc`，和WebRTC做法一致，因此同样的参考WebRTC，我们通过`GetModuleHandleExA`来获取对应的`hInstance`，即

```cpp
HMODULE hInstance = nullptr;
DWORD flags = GET_MODULE_HANDLE_EX_FLAG_FROM_ADDRESS |
              GET_MODULE_HANDLE_EX_FLAG_UNCHANGED_REFCOUNT;
CHECK_RET(GetModuleHandleExA(
        flags, reinterpret_cast<char *>(&DefWindowProc), &hInstance));
```

但排查过程中找到一篇[What is the HINSTANCE passed to CreateWindow and RegisterClass used for?](https://devblogs.microsoft.com/oldnewthing/20050418-59/?p=35873)，详细解释了hInstance的作用。hInstance实际是用来区分不同模块中可能相同名称的窗口类的，因此猜测这里应该传入当前使用的窗口类所在的模块句柄。我们这里使用的是自己创建的窗口类，因此应该传入当前模块的句柄。于是在当前模块中实现自定义的窗口过程函数，并通过其地址获取当前模块句柄，将这两个值作为参数传入，问题解决。

```cpp
void Init{
    ...
    HMODULE hInstance = nullptr;
    DWORD flags = GET_MODULE_HANDLE_EX_FLAG_FROM_ADDRESS |
                  GET_MODULE_HANDLE_EX_FLAG_UNCHANGED_REFCOUNT;
    CHECK_RET(GetModuleHandleExA(
            flags, reinterpret_cast<char *>(&WndProc),
            &hInstance));

    WNDCLASSEXW wcex = {};
    wcex.cbSize = sizeof(WNDCLASSEXW);
    wcex.lpfnWndProc = WndProc;
    wcex.hInstance = hInstance;
    wcex.hCursor = LoadCursor(nullptr, IDC_ARROW);
    wcex.lpszClassName = kMagnifierHostClass;
    CHECK_RET(RegisterClassExW(&wcex));

    ...
}

static LRESULT CALLBACK WndProc(HWND hwnd, UINT uMsg, WPARAM wParam,
                                LPARAM lParam) {
    return DefWindowProc(hwnd, uMsg, wParam, lParam);
}

```

实测后发现`WNDCLASSEX`结构体中的`lpfnWndProc`设置为`DefWindowProc`，而`hInstance`设置为当前类所在模块依然可以初始化成功，进一步说明至少在Win7下，`hInstance`并不表示窗口过程函数所在模块，而是窗口类所在模块。但由于Win10/11下并不存在问题，因此并不清楚是Win7下接口实现存在bug，还是官方文档有错误，最好的办法是将窗口过程函数和窗口类定义在同一模块，这样就绝对没有风险了。
