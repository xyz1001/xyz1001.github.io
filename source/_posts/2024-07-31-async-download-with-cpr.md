---
title: 踩坑记之基于cpr实现异步下载
author: 张帆
tags:
  - 网络
  - C++
date: 2024-07-31 19:47:22
---

cpr是一个基于libcurl的C++封装库，提供了非常便于使用的上层接口。对于大量数据的传输操作，其提供了同步和异步两种接口接口。

``` cpp
    Response Download(const WriteCallback& write);
    AsyncResponse DownloadAsync(const WriteCallback& write);
```
其中，`AsyncResponse`是一个类似`std::future`的对象，应用需要在 **合适的时机** 调用`get()`方法获取`Response`对象。

在业务开发中，我们需要实现异步下载功能，函数签名如下：
``` cpp
    void DownloadAsync(const std::string& url, const std::string& file_path, std::function<void(bool)> callback);
```
应用在主线程调用该方法，传输下载链接和下载文件路径，并提供一个回调在下载完成/出错时进行回调。

<!--more-->

## 基于异步接口实现

首先我们自然的会考虑DownloadAsync方法。一个有问题的实现如下：

``` cpp
std::ofstream fout(path, std::ios::binary);
// cpr::DownloadAsync内部会调用std::shared_from_this()获取shared_ptr<cpr::Session>对象，因此cpr::Sessiond对象需要是shared_ptr
auto session = std::make_shared<cpr::Session>();
session->SetUrl({url});

auto write_data = [this, &fout](std::string data, intptr_t /*userdata*/) {
    fout.write(data.data(), data.size());
    return true;
};
auto async_response =
        session->DownloadAsync(cpr::WriteCallback{write_data});
auto response = async_response.get();
callback(response.error.code == cpr::ErrorCode::OK);
```

这里的问题在于，我们在主线程中调用了get()方法，这会导致主线程阻塞，直到下载完成。这样的实现与同步接口没有区别，无法发挥异步接口的优势。但如果不调用get()方法，我们就无法获取下载结果。
因此，我们需要在下载完成时，由cpr库调用我们的回调函数。这里我们可以借助cpr的下载进度回调，修改如下：

``` cpp
auto async_response = std::make_shared<std::optional<cpr::AsyncResponse>>();
auto progress_callback = [this, async_response, callback](
                                 cpr::cpr_pf_arg_t total,
                                 cpr::cpr_pf_arg_t now, auto...) {
    if (total == 0) {
        return true;
    }
    if (now == total) {
        auto response = (*async_response)->get();
        callback(response.error.code == cpr::ErrorCode::OK);
    }
    return true;
};
session->SetProgressCallback({progress_callback});

auto write_data = [this, fout](std::string data, intptr_t /*userdata*/) {
    fout->write(data.data(), data.size());
    return true;
};
*async_response = session->DownloadAsync(cpr::WriteCallback{write_data});
```

这里由于需要在设置进度回调后，再去将`AsyncResponse`对象写入lambda捕获列表中，同时`AsyncResponse`又不支持默认构造，因此只能使用`std::shared_ptr<std::optional<cpr::AsyncResponse>>`这种方式实现。但即使这样修改后，依然无法使用。get()方法会阻塞cpr的回调线程，导致下载进度回调无法返回，而`cpr::AsyncResponse::get()`必须又需要在进度回调返回后才会返回，这样就陷入了死锁。同时，我们无法获取出错时的信息，cpr并没有提供错误回调。因此，cpr提供的异步接口看上去很美好，但由于很难找到一个合适的时机调用get()方法，因此很难使用。

## 基于同步接口实现

接下来我们只能考虑使用同步接口。但我们不能在主线程中调用同步接口，因为这会导致主线程阻塞。因此我们需要在一个新的线程中调用同步接口。

``` cpp
if (!running_.exchange(true)) {
    thread_ = std::thread([this, url = std::string(url),
                           path = std::move(path),
                           callback = std::move(callback)]() {
        std::ofstream fout(path, std::ios::binary);
        cpr::Session session;
        session.SetUrl({url});

        auto write_data = [this, &fout](std::string data,
                                        intptr_t /*userdata*/) {
            fout.write(data.data(), data.size());
            return true;
        };
        auto response = session.Download(cpr::WriteCallback{write_data});
        fout.close();
        callback(response.error.code == cpr::ErrorCode::OK);
    });
}
```

但同步接口的问题在于，我们需要能够中断该操作，若下载未完成时析构下载对象，不能长时间阻塞。首先想到的是可以在写回调中检测running_成员变量，当Downloader对象析构时会将该变量置为false。但由于写回调是在通过网络接收到数据时才会调用，一旦因为网络问题导致长时间无法接收数据，我们就无法及时中断下载。cpr提供了另一个用于通知下载/上传京都的回调[ProgressCallback](https://docs.libcpr.org/advanced-usage.html#asynchronous-callbacks)，这个函数也可以通过返回false来中断下载/上传。查看cpr的源码，该回调对应libcurl选项是[CURLOPT_PROGRESSFUNCTION](https://curl.se/libcurl/c/CURLOPT_PROGRESSFUNCTION.html)，libcurl文档中明确保证该接口至少会每秒调用一次。因此我们可以在ProgressCallback中检测running_变量，来实现中断下载。这样即使当前网络不通，我们也能在1秒内中断下载，对于低频操作+低频场景而言，虽然还是有可能会阻塞主线程，但相对可以接受，已经足以解决问题。最终的实现如下：

``` cpp
Downloader::~Downloader() {
    if(running_.exchange(false)) {
        thread_.join();
    }
}

void Downloader::DownloadAsync(std::string_view url, std::filesystem::path path,
                               std::function<void(bool)> callback) {
    if (!running_.exchange(true)) {
        thread_ = std::thread([this, url = std::string(url),
                               path = std::move(path),
                               callback = std::move(callback)]() {
            std::ofstream fout(path, std::ios::binary);
            cpr::Session session;
            session.SetUrl({url});

            session.SetProgressCallback(
                    {[this](auto...) { return running_.load(); }});

            auto write_data = [this, &fout](std::string data,
                                            intptr_t /*userdata*/) {
                fout.write(data.data(), data.size());
                return true;
            };
            auto response = session.Download(cpr::WriteCallback{write_data});
            bool ok = response.error.code == cpr::ErrorCode::OK;
            if(!ok) {
                fout.close();
                std::filesystem::remove(path);
            }
            callback(ok);
        });
    }
}
```
