---
layout: post
title: 一篇就弄懂Linux netstat和tcpdump的常见用法
tags:
  - Linux
  - 运维实施
permalink: /posts/:title/
comments: true
---

* 目录
{:toc}

## 1. Netstat
Netstat是一款CLI工具，它可以列出系统上所有的网络连接情况，包括tcp、udp和其他unix网络socket，另外它还能列出处于监听状态的socket。当Linux网络或系统排查问题时，netstat基本上是必用的工具之一，下面开始对netstat的常见用法加以说明。
### netstat -a
用途：列出tcp、udp和其他unix套接字下所有的连接，往往信息提供的不够直观和详细，因此也常通过搭配其他参数一起使用。

* 比如仅列出tcp或者udp协议下连接：`netstat -at 或者 netstat -au`，如下图所示列出了所有的tcp连接：
![](https://upload-images.jianshu.io/upload_images/12911861-38908cb53bcb07a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)
### 禁用反向域名解析
相信很多用户已经发现，在执行上述命令的时候结果查询的不是很快，这是因为netstat默认会使用反向域名解析，会把对应的IPv4地址解析对应到IPv6或者主机名，如果IP地址信息足够满足需求时，大可以禁用反向域名解析，加上`-n`参数。

* 比如`netstat -ant`查询所有tcp协议下的连接：
![](https://upload-images.jianshu.io/upload_images/12911861-ec26fe3880d9c223.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)
### 查看处于监听状态的连接
* 比如查看处于监听状态的所有tcp连接：`netstat -nlt`
![](https://upload-images.jianshu.io/upload_images/12911861-87598b9b6e622ce1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

如果是查看udp协议下的连接，则用`netstat -nlu`。
### 查看连接对应的程序信息
常常会遇到如下场景：我们提前预知某服务程序开放某监听端口，于是想查看对应连接的信息，包括进程名、进程ID甚至执行owner信息等，可加入`-p`，`-e`参数来实现。
* 比如查看端口3307对应的服务进程信息：
![](https://upload-images.jianshu.io/upload_images/12911861-9971647ed154a532.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)
### 其它
`netstat -s`可以打印出网络统计数据，包括某个协议下的收发包数量；
`netstat -rn`可查看路由信息，效果等同于`route -n`；
`netstat -i`或者`netstat -ie`可查看网络接口信息，且后一条命令效果等同于`ifconfig -a`；

## 2. TCPDump
用简单的话来定义tcpdump，就是：`dump the traffic on a network`，根据使用者的定义对网络上的数据包进行截获的包分析工具。 tcpdump可以将网络中传送的数据包的“头”完全截获下来提供分析。它支持针对网络层、协议、主机、网络或端口的过滤，并提供and、or、not等逻辑语句来帮助你去掉无用的信息，是一个十分常用的Linux抓包工具。

在介绍tcpdump常用命令之前先提一个小建议，每个tcpdump命令的末尾加上`-s 0 -nvvvtttt`参数，简单解释一下参数的含义：
* `-s 0`：-s参数是设置tcpdump抓取数据包的长度，如果不设置则默认是68字节，而设置为0 意味着让tcpdump自动选择合适的长度来抓取数据包；
* `-n`：不对地址(比如, 主机地址, 端口号)进行数字表示到名字表示的转换；
* `-vvv`：输出详细的最高级别；
* `-tttt`：每行打印数据的开头添加日期样式（Y-M-D h:m:s）;

另外还可以结合自己的需要选择要不要加上`-x -xx -X -XX`（输出会打印每个包的头部数据，且会以16进制或者ASCII码形式打印每个包的数据）等参数

### 抓取指定网络接口的数据包
```bash
# 绑定网卡，命令中br-mgmt是对应的网络接口
tcpdump -i br-mgmt -nvvvtttt
```
### 抓取指定主机的数据包
* 抓取主机名是node-6.domain.tld收到或发出的数据包
```bash
tcpdump host node-6.domain.tld -nvvvtttt
# 也可以把主机名换成ip地址，比如：
tcpdump host 10.20.0.9 -nvvvtttt
```
* 抓取主机间通信的数据包
```bash
# 抓取主机node-1与node-2之间的通信的数据包
tcpdump host node-1 and node-2 -nvvvtttt
# 抓取主机node-1与node-2或node-3之间的通信的数据包
tcpdump host node-1 and \(node-2 or node-3\) -nvvvtttt
# 抓取主机node-1的所有的收到或者发出数据通信包，除了与node-2间的通信
tcpdump host node-1 and not node-2 -nvvvtttt
# 以上命令主机名都可以换成ip地址，但注意在非条件下，not -->！
```
* 另外还可以指定源主机（ip地址）和目标主机（ip地址）
```bash
# 截获主机node-1发出的数据包
tcpdump -i br-mgmt src host node-1 -nvvvtttt
# 截获主机node-1收到的数据包
tcpdump -i br-mgmt dst host node-1 -nvvvtttt
```
### 抓取指定主机和指定端口的数据包
* 抓取主机node-1：5673所有发出或收到的TCP数据包：
```bash
tcpdump tcp -i br-mgmt port 5673 and host node-1 -nvvvtttt 
```
* 抓取主机node-1：4953所有发出或收到的UDP数据包：
```bash
tcpdump udp -i br-mgmt port 4953 and host node-1 -nvvvtttt
```
### 一个比较详细的例子
```bash
tcpdump tcp -i br-mgmt -s 0 -c 100 and dst port ! 22 and src host 192.168.0.6 -nvvvtttt -X -w ./mgnt.cap
```
如下仅简单说明一下上文没提到的用法：
* `-c 100`：抓取100个符合过滤条件的数据包；
* `-w ./mgmt.cap`：将抓取的数据包保存到mgmt.cap文件，可以配合Windows Wireshark分析抓取的数据包；
### 抓取HTTP数据包
* tcpdump过滤HTTP的GET请求:
```bash
tcpdump -s 0 -A 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420' -Xnvvvtttt
```
* tcpdump过滤HTTP的POST请求:
```bash
tcpdump -s 0 -A 'tcp dst port 80 and (tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354)' -Xnvvvtttt
```
