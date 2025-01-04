---
title: 基于图算法重构libyuv图像转换器
author: 张帆
tags:
  - 音视频
  - 算法
  - C++
date: 2024-12-28 15:12:15
---

## 问题背景

libyuv是一个google开发的用于进行原始图像数据转换的C++库，其提供了对多种格式（YUV/RGB）图像的多种转换操作（旋转/缩放/镜像/裁剪/格式转换），目前在内部业务中存在一个图像处理器模块，用于基于libyuv进行原始图像数据的处理，如旋转/缩放/格式转换。每新增一种输入输出的支持，需要编写对应的转换函数，很多转换都需要多次调用libyuv接口，其中大部分都是类似的逻辑，差异点主要在于调用的libyuv接口和参数有差异。以NV12转换成BGRA为例，有如下的转换流程图：

![转换流程图](libyuv_convert.png)

可以看到，每新增一种转换格式支持，一方面需要编写大量代码，另一方面需要自行寻找可用的转换路径，libyuv并没有提供任意格式的转换函数，例如NV12Rotate函数是没有提供现成的，因此只能先转换成I420再旋转或基于更底层API实现一个NV12Rotate。在需要支持的输入输出参数种类较少时，手动编写转换步骤代码还可以接受。但随着业务的需要，需要支持的输入输出格式越来越多，目前仅像素格式（pixel format）需要支持的就多达数十种，再考虑到旋转，缩放等，代码迅速膨胀，维护成本也随之迅速上升。

<!--more-->

## 解决思路

libyuv提供的接口，无论是哪种转换操作，都可以看成一个转换关系，如果我们将libyuv提供的转换函数的输入输出格式看成点，转换函数看成边，最终实际上组成了一个有向图，也就是说，我们的问题可以转换成给定输入输出点，在一个有向图中寻找出最短路径。这样我们只需要将有向图的边整理出来，对于任意的输入输出格式都可以使用统一的寻找最短路径再执行的逻辑进行处理。


## 确定数据结构

首先我们需要确定顶点和边对应的数据结构。在当前的业务场景中，我们主要使用libyuv完成旋转/缩放/格式转换的操作，因此输入输出格式会在这几方面存在差异，我们的顶点数据结构中需要包含这些差异项，同时格式转换还需要区分颜色空间（额外约束）。顶点数据结构定义如下

``` cpp
struct Node {
    FourCC format;
    bool rotate;
    bool scale;
    ColorRange color_range;
};
```

其中`format`表示数据格式，`rotate`表示当前顶点相对于原始数据是否发生旋转，`scale`表示当前顶点对于原始数据是否发生缩放。`color_range`表示颜色空间（`Full`和`Limit`）。

对于边，代表libyuv转换函数，首先不同的转换函数的开销不同，因此这里边是有权重的，另一方面部分转换操作可能没有合适的转换函数可用，这时只能通过有损方式进行转换，如`I422Scale`方法是没有的，我们只能先有损转换I422到I420，再调用`I420Scale`。我们希望有损转换尽量不要使用，因此可以将有损转换的开销提高。边的数据结构定义如下：

``` cpp
struct Edge {
    Node from;
    Node to;
    int cost;
    void *func;
    const char *func_name;
};
```

这里由于libyuv转换函数的原型不统一，所以只能将类型擦除，使用`void *`保存，后续会处理如何还原并调用的问题。


## 最短路径搜索算法

结合上述分析，我们要实现的算法是在一个有向有环带权图中，找到一条最短路径。

``` cpp
using Node = int;
constexpr const int kMaxCost = 99999;
struct Edge {
    Node from;
    Node to;
    int cost;
};
 
class ShortestPathSolver {
public:
    ShortestPathSolver(const std::vector<Edge> &edges) {
        for (const auto &edge : edges) {
            road_map_[edge.to].push_back(edge.from);
            distance_[{edge.from, edge.to}] = edge.cost;
        }
    }
 
public:
    struct Result {
        std::vector<Node> path;
        int total_cost;
    };
 
    Result GetShortestPath(Node start, Node end) {
        std::map<std::pair<Node, Node>, int> shortest_distance;
        std::map<Node, Node> shortest_pre;
        std::map<Node, bool> visited;
        int total_cost =
                Dfs(start, end, shortest_distance, shortest_pre, visited);
        if (total_cost == kMaxCost) { return {{}, total_cost}; }
        return {ShortedRoad(start, end, shortest_pre), total_cost};
    }
 
private:
    int Dfs(Node start, Node end,
            std::map<std::pair<Node, Node>, int> &shortest_distance,
            std::map<Node, Node> &shortest_pre, std::map<Node, bool> visited) {
 
        if (start == end) { return 0; }
        if (visited[end]) { return kMaxCost; }
 
        if (shortest_distance.count({start, end})) {
            return shortest_distance[{start, end}];
        }
 
        int min_dis = kMaxCost;
        Node min_pre = end;
        for (auto &pre : road_map_[end]) {
 
            visited[end] = true;
            auto cur_dis =
                    Dfs(start, pre, shortest_distance, shortest_pre, visited) +
                    distance_[{pre, end}];
            visited[end] = false;
 
            if (cur_dis < min_dis) {
                min_dis = cur_dis;
                min_pre = pre;
            }
        }
        shortest_distance[{start, end}] = min_dis;
        shortest_pre[end] = min_pre;
        return min_dis;
    }
 
    std::vector<Node> ShortedRoad(Node start, Node end,
                                  std::map<Node, Node> shortest_pre) {
        Node curr = end;
        std::vector<Node> ret = {end};
 
        while (curr != start) {
            curr = shortest_pre[curr];
            ret.push_back(curr);
        }
 
        std::reverse(ret.begin(), ret.end());
        return ret;
    }
 
private:
    std::map<Node, std::vector<Node>> road_map_;
    std::map<std::pair<Node, Node>, int> distance_;
};
```


## 枚举可用边

根据上一步中的算法实现，我们需要提供一个边的集合作为参数传入。这里遇到的问题是libyuv提供的转换函数较多，同时部分转换函数可能对顶点中的若干个参数不关心，也就是可以同时作用于多个不同顶点的转换，即对应多条边。例如`I420Rotate`函数，其不关心`scale`和`color_range`参数，因此可以支持`Node{format: I420, scale: false, rotate: false, color_range: Full}`到`Node{format: I420, scale: false, rotate: true, color_range: Full}`，对应一条边，也支持`Node{format: I420, scale: true, rotate: false, color_range: Full}`到`Node{format: I420, scale: true, rotate: true, color_range: Full}`，对应另一条边。因此要枚举所有的边工作量非常之大，这里我们一方面仅添加我们当前需要使用的转换函数，另一方面，考虑到当前使用的操作仅旋转/缩放/格式转换三种，且每一种对应的函数关心的参数是一样的，因此通过工具函数批量生成边。

``` cpp
void AppendRotateEdges(FourCC fourcc, int cost, void *func,
                       const char *func_name, std::vector<Edge> &edges) {
    for (auto &scale : {true, false}) {
        for (auto &color_range : {ColorRange::kFull, ColorRange::kLimited}) {
            edges.emplace_back(Node(fourcc, false, scale, color_range),
                               Node(fourcc, true, scale, color_range), cost,
                               func, func_name);
            edges.emplace_back(Node(fourcc, true, scale, color_range),
                               Node(fourcc, false, scale, color_range), cost,
                               func, func_name);
        }
    }
}
 
void AppendScaleEdges(FourCC fourcc, int cost, void *func,
                      const char *func_name, std::vector<Edge> &edges) {
    for (auto &rotate : {true, false}) {
        for (auto &color_range : {ColorRange::kFull, ColorRange::kLimited}) {
            edges.emplace_back(Node(fourcc, rotate, false, color_range),
                               Node(fourcc, rotate, true, color_range), cost,
                               func, func_name);
            edges.emplace_back(Node(fourcc, rotate, true, color_range),
                               Node(fourcc, rotate, false, color_range), cost,
                               func, func_name);
        }
    }
}
 
void AppendConvertEdges(FourCC from, FourCC to, int cost, void *func,
                        const char *func_name, std::vector<Edge> &edges) {
    for (auto &rotate : {true, false}) {
        for (auto &scale : {true, false}) {
            for (auto &color_range :
                 {ColorRange::kFull, ColorRange::kLimited}) {
                edges.emplace_back(Node(from, rotate, scale, color_range),
                                   Node(to, rotate, scale, color_range), cost,
                                   func, func_name);
            }
        }
    }
}
 
void AppendConvertEdges(FourCC from_fourcc, ColorRange from_color_range,
                        FourCC to_fourcc, ColorRange to_color_range, int cost,
                        void *func, const char *func_name,
                        std::vector<Edge> &edges) {
    for (auto &rotate : {true, false}) {
        for (auto &scale : {true, false}) {
            edges.emplace_back(
                    Node(from_fourcc, rotate, scale, from_color_range),
                    Node(to_fourcc, rotate, scale, to_color_range), cost, func,
                    func_name);
        }
    }
}
 
#define FUNCTION(func) reinterpret_cast<void *>(func), #func
 
std::vector<Edge> MakeEdges() {
    std::vector<Edge> edges;
    AppendRotateEdges(kBGRA, 1, FUNCTION(libyuv::ARGBRotate), edges);
    ...
    AppendRotateEdges(kI444, 1, FUNCTION(libyuv::I444Rotate), edges);
 
    AppendScaleEdges(kBGRA, 1, FUNCTION(libyuv::ARGBScale), edges);
    ...
    AppendScaleEdges(kI444, 1, FUNCTION(libyuv::I444Scale), edges);
 
    AppendConvertEdges(kBGRA, ColorRange::kFull, kNV12, ColorRange::kLimited, 2,
                       FUNCTION(libyuv::ARGBToNV12), edges);
    ...
    AppendConvertEdges(kI444, ColorRange::kLimited, kRGB565, ColorRange::kFull,
                       2, FUNCTION(libyuv::I444ToRGB24), edges);
    return edges;
}
```

这样就完成了边的集合的创建。后续若有新的输入输出格式需要支持，只需要添加边即可。

## 边的执行

完成以上步骤后，我们就可以得到一条转换路径（当前使用边的数据表示），我们只需要依次执行边结构体中的函数就可以了。这里的问题在于我们之前将函数类型擦除了，要如何确定调用参数的类型，个数和位置？考虑到libyuv的函数做且仅作一件事，且相同操作的libyuv函数原型是非常类似的。以缩放操作为例，无论是`I420Scale`还是`ARGBScale`，其函数原型都遵循以下格式

``` cpp
int 函数名(输入画面平面0数据指针,
              输入画面平面0行宽,
              输入画面平面1数据指针,
              输入画面平面2行宽,
              ...,
              输入画面宽度,
              输入画面高度,
              输出画面平面0数据指针,
              输出画面平面0行宽,
              输出画面平面1数据指针,
              输出画面平面2行宽,
              ...,
              输出画面宽度,
              输出画面高度,
              缩放策略);
```

因此我们首先可以根据`from`和`to`顶点中参数的差异确定函数所执行操作的类型。然后再根据输入输出格式得到对应平面数量，这样我们就能确定函数原型。按照这个思路，我们可以写出如下代码

``` cpp
std::shared_ptr<HostPixmap> LibyuvProcessor::Convert(
        std::shared_ptr<HostPixmap> from, std::shared_ptr<HostPixmap> to) {
    auto from_node = Node(from->GetPixelFormat(),
                          from->GetRotation() != Rotation::kRotate0, false,
                          from->GetColorRange());
    auto to_node =
            Node(to->GetPixelFormat(), to->GetRotation() != Rotation::kRotate0,
                 to->GetOriginalSize() != from->GetOriginalSize(),
                 to->GetColorRange());
    if (convert_path_.empty() || convert_path_.front().from != from_node ||
        convert_path_.back().to != to_node) {
        convert_path_ =
                shortest_path_solver_.GetShortestPath(from_node, to_node);
        if (convert_path_.empty()) {
            LOGE("No convert path from {} to {}", std::string(from_node),
                 std::string(to_node));
            return nullptr;
        }
        std::string path_desc;
        for (auto &i : convert_path_) {
            path_desc += fmt::format("[{}] --{}--> ", std::string(i.from),
                                     i.func_name);
        }
        path_desc += fmt::format("[{}]", std::string(convert_path_.back().to));
        LOGI("Convert path: {}", path_desc);
    }
 
    std::shared_ptr<HostPixmap> current = from;
    for (auto &i : convert_path_) {
        std::shared_ptr<HostPixmap> buffer = to;
        if (i.from.rotate != i.to.rotate) {
            if (i.to != to_node) {
                buffer =
                        VectorPixmap::Builder(resource_)
                                .ShallowCopy(*current)
                                .Rotate(to->GetRotation() - from->GetRotation())
                                .Build();
            }
            auto rotation = buffer->GetRotation() - current->GetRotation();
            auto src = current->GetPlanes();
            auto dst = buffer->GetPlanes();
            auto plane_count = GetPlaneCount(i.from.format);
            switch (plane_count) {
            case 1:
                reinterpret_cast<void (*)(const uint8_t *, int, int, int,
                                          uint8_t *, int, int, int,
                                          libyuv::FilterMode)>(i.func)(
                        src[0].data, src[0].linesize,
                        current->GetRotatedSize().w,
                        current->GetRotatedSize().h, dst[0].data,
                        dst[0].linesize, buffer->GetRotatedSize().w,
                        buffer->GetRotatedSize().h,
                        libyuv::FilterMode::kFilterBox);
                break;
            case 2:
                reinterpret_cast<void (*)(const uint8_t *, int,const uint8_t *, int, int, int,
                                          uint8_t *, int,uint8_t *, int, int, int,
                                          libyuv::FilterMode)>(i.func)(
                        src[0].data, src[0].linesize,
                        src[1].data, src[1].linesize,
                        current->GetRotatedSize().w,
                        current->GetRotatedSize().h, dst[0].data,
                        dst[0].linesize,
                        dst[1].data, dst[1].linesize,
                        buffer->GetRotatedSize().w,
                        buffer->GetRotatedSize().h,
                        libyuv::FilterMode::kFilterBox);
                break;
            case 3:
                reinterpret_cast<void (*)(const uint8_t *, int,const uint8_t *, int,const uint8_t *, int, int, int,
                                          uint8_t *, int,uint8_t *, int,uint8_t *, int, int, int,
                                          libyuv::FilterMode)>(i.func)(
                        src[0].data, src[0].linesize,
                        src[1].data, src[1].linesize,
                        src[2].data, src[2].linesize,
                        current->GetRotatedSize().w,
                        current->GetRotatedSize().h, dst[0].data,
                        dst[0].linesize,
                        dst[1].data, dst[1].linesize,
                        dst[2].data, dst[2].linesize,
                        buffer->GetRotatedSize().w,
                        buffer->GetRotatedSize().h,
                        libyuv::FilterMode::kFilterBox);
                break;
            default:
                LOGC("Not support plane count: {}", plane_count);
                return nullptr;
            }
            return to;
        } else if (i.from.scale != i.to.scale) {
            if (i.to != to_node) {
                buffer =
                        VectorPixmap::Builder(resource_)
                                .ShallowCopy(*current)
                                .SetSize(to->GetRotatedSize())
                                .Rotate(from->GetRotation() - to->GetRotation())
                                .Build();
            }
            // TODO
        } else {
            if (i.to != to_node) {
                buffer = VectorPixmap::Builder(resource_)
                                 .ShallowCopy(*current)
                                 .SetPixelFormat(to->GetPixelFormat())
                                 .SetColorRange(to->GetColorRange())
                                 .Build();
            }
            // TODO
        }
    }
    return to;
}
```


## 调用优化

写了部分代码后发现这样还是太繁琐，每次都要手写函数原型。首先这里可以通过变参模板做一些简化

``` cpp
template <typename... ARGS>
int InvokeLibyuvMethod(void *func, ARGS... args) {
    return reinterpret_cast<int (*)(ARGS...)>(func)(args...);
}
 
std::shared_ptr<HostPixmap> LibyuvProcessor::Convert(
        std::shared_ptr<HostPixmap> from, std::shared_ptr<HostPixmap> to) {
    ...
    for (auto &i : convert_path_) {
        std::shared_ptr<HostPixmap> buffer = to;
        if (i.from.rotate != i.to.rotate) {
            ...
            switch (plane_count) {
            case 1:
                InvokeLibyuvMethod(i.func,
                        src[0].data, src[0].linesize,
                        current->GetRotatedSize().w,
                        current->GetRotatedSize().h, dst[0].data,
                        dst[0].linesize, buffer->GetRotatedSize().w,
                        buffer->GetRotatedSize().h,
                        libyuv::FilterMode::kFilterBox);
                break;
        ...
}
```

这里相对来说还是比较麻烦，需要填写一串参数，容易出错。但实际上三个`case`中仅仅是`data`和`linesize`个数有差异，这种情况 很容易想到可以通过非类型模板参数和可变参数模板展开来实现，很容易写出以下代码

``` cpp
template <size_t... Indices>
int InvokeLibyuvRotateMethod(std::index_sequence<Indices...>, void *func,
                             std::array<HostPixmap::Plane, 4> src,
                             std::array<HostPixmap::Plane, 4> dst, Size size,
                             Rotation rotation) {
    InvokeLibyuvMethod(func, src[Indices].data..., src[Indices].linesize...,
                       size.w, size.h, dst[Indices].data..., dst[Indices].linesize...,
                       size.w, size.h, Map(rotation), libyuv::FilterMode::kFilterBox);
}
 
template <int T, typename... ARGS>
int InvokeLibyuvRotateMethod(ARGS... args) {
    return InvokeLibyuvRotateMethod(std::make_index_sequence<T>(), args...);
}
```

这里先通过非类型模板参数，将平面个数T传入，然后为了将数组解包，我们通过`std::make_index_sequence`将`T`转换成下标序列，再通过可变参数模板展开。但这里的问题是libyuv要求的参数顺序是`data`和`linesize`交错，而这里只能实现连续的`data`和连续的`linesize`。为了实现`data`和`linesize`交错展开，我查阅了大量资料，也没有找到办法，在这里卡住了一上午。

天无绝人之路，当我意识到无法通过可变参模板展开实现时，突然意识到支持参数解包还有一种途径是调用`std::apply`。我们只需要先将参数组装成一个`tuple`即可。最终实现代码如下：

``` cpp
template <typename... ARGS>
auto ExpandArgs(ARGS... args) {
    return std::tuple_cat([](auto arg) {
        if constexpr (std::is_same_v<decltype(arg), HostPixmap::Plane>) {
            return std::make_tuple(arg.data, arg.linesize);
        } else {
            return std::make_tuple(arg);
        }
    }(args)...);
}
 
template <size_t... Indices>
int InvokeLibyuvRotateMethod(std::index_sequence<Indices...>, void *func,
                             std::array<HostPixmap::Plane, 4> src,
                             std::array<HostPixmap::Plane, 4> dst, Size size,
                             Rotation rotation) {
    auto args = ExpandArgs(src[Indices]..., dst[Indices]..., size.w, size.h,
                           Map(rotation));
    return std::apply(
            [func](auto... args) { return InvokeLibyuvMethod(func, args...); },
            args);
}
 
std::shared_ptr<HostPixmap> LibyuvProcessor::Convert(
        std::shared_ptr<HostPixmap> from, std::shared_ptr<HostPixmap> to) {
    ...
    for (auto &i : convert_path_) {
        std::shared_ptr<HostPixmap> buffer = to;
        if (i.from.rotate != i.to.rotate) {
            ...
                switch (plane_count) {
            switch (plane_count) {
            case 1:
                InvokeLibyuvRotateMethod<1>(
                        i.func, src, dst, current->GetRotatedSize(), rotation);
                break;
            case 2:
                InvokeLibyuvRotateMethod<2>(
                        i.func, src, dst, current->GetRotatedSize(), rotation);
                break;
            case 3:
                InvokeLibyuvRotateMethod<3>(
                        i.func, src, dst, current->GetRotatedSize(), rotation);
                break;
            ...
    }
    ...
}
```

这里我们先将数组直接解包成指定个数参数，但并不展开成员变量，然后再调用`ExpandArgs`将类型为`Plane`结构体的参数做一次成员变量展开，并使用`tuple_cast`将展开的参数元组进行拼接。这里使用到了C++17的折叠表达式和`constexpr if`来实现编译期展开。最后，再将参数元组使用`std::apply`进行再次展开，这里由于`InvokeLibyuvMethod`是一个模板参数，为了避免编写一长串参数类型，这里通过变参`lambda`进行了一个包装。

## 总结

至此，我们就完成了整个的重构。重构后，我们不用再一个个针对具体的类型编写繁琐的转换过程，只需要在构建有向图时添加新的边即可，新增输入输出格式时，至多只需要添加一两行代码即可。
