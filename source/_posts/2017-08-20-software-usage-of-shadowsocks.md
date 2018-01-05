---
title: 软件折腾笔记之Shadowsocks
author: 张帆
tags:
  - 软件折腾笔记
  - shadowsocks
date: 2017-08-20 23:44:14
---

## 服务器配置

### 安装环境

``` bash
sudo apt install python3-pip
sudo pip3 install shadowsocks
```

### 优化系统配置

#### 开启TCP BBR拥塞控制算法

  1. 且服务器虚拟化方式为xen或kvm，TCP BBR不支持OpenVZ，检测方式：
  `sudo apt install virt-what && sudo virt-what`
  2. 检查内核是否为4.9以上，否则自行升级内核至4.9以上或更换较新的系统
  3. 开启BBR
  ```
  echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
  echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
  sysctl -p
  ```
  输入`sysctl net.ipv4.tcp_available_congestion_control`检查是否开启成功
  输入`lsmod | grep bbr`检查是否加载成功

 <!--more-->

#### 优化网络配置

 在`/etc/sysctl.d/local.conf`中添加以下内容
 ```
 # max open files
 fs.file-max = 51200
 # max read buffer
 net.core.rmem_max = 67108864
 # max write buffer
 net.core.wmem_max = 67108864
 # default read buffer
 net.core.rmem_default = 65536
 # default write buffer
 net.core.wmem_default = 65536
 # max processor input queue
 net.core.netdev_max_backlog = 4096
 # max backlog
 net.core.somaxconn = 4096
 # resist SYN flood attacks
 net.ipv4.tcp_syncookies = 1
 # reuse timewait sockets when safe
 net.ipv4.tcp_tw_reuse = 1
 # turn off fast timewait sockets recycling
 net.ipv4.tcp_tw_recycle = 0
 # short FIN timeout
 net.ipv4.tcp_fin_timeout = 30
 # short keepalive time
 net.ipv4.tcp_keepalive_time = 1200
 # outbound port range
 net.ipv4.ip_local_port_range = 10000 65000
 # max SYN backlog
 net.ipv4.tcp_max_syn_backlog = 4096
 # max timewait sockets held by system simultaneously
 net.ipv4.tcp_max_tw_buckets = 5000
 # turn on TCP Fast Open on both client and server side
 net.ipv4.tcp_fastopen = 3
 # TCP receive buffer
 net.ipv4.tcp_rmem = 4096 87380 67108864
 # TCP write buffer
 net.ipv4.tcp_wmem = 4096 65536 67108864
 # turn on path MTU discovery
 net.ipv4.tcp_mtu_probing = 1
 ```
### 编写配置文件

 新建配置文件`/etc/shadowsocks.json`，添加以下内容
 ```
 {
     "server":"my_server_ip",
     "server_port": 433
     "local_address": "127.0.0.1",
     "local_port":1080,
     "password":"mypassword",
     "timeout":300,
     "method":"aes-256-cfb",
     "fast_open": true
 }
 ```

### 运行/停止程序

 - `ssserver -c /etc/shadowsocks.json -d start`
 - `ssserver -c /etc/shadowsocks.json -d stop`

### NOTE

- 服务器系统推荐选择具有较新内核(`>=4.9`)的系统，避免升级内核步骤，推荐使用`Ubuntu`，直接支持`tcp fast_open`
- 选购服务器时注意根据自身的地理位置、网络状况来选择合适的服务器提供商，测试指标一般有网速和ping延迟，推荐工具有`traceroute`，`奇云测`，`服务器提供商官方测速地址`

## 客户端配置

1. 下载GUI客户端
2. 添加服务器信息

### NOTE

- 搭配各类软件可实现shadowsocks的多协议、多场景支持，部分如下

|软件|协议或场景|
|---|---|
|proxychains4|命令行|
|SwitchyOmega|chrome浏览器|
|privoxy|sock5转http|

- 首次使用后推荐到处配置信息为`URI`并保存，方便下次使用时无需输入配置信息
