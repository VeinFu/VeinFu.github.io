---
layout: post
title: 一篇就弄懂WSGI、uwsgi和uWSGI的区别
category: 技术博客
tags:
  - 一篇就弄懂系列
  - 运维实施
comments: true
---

* 目录
{:toc}

## 引言
最近基于Django Web框架在开发一个后端项目，在Web Server和Django应用程序交互的过程中总会碰到本文题目提及到的几个概念，笔者特意花了点时间研究了下，为方便以后温习特在此记录一下。

## WSGI
英文全称：Web Server Gateway Interface，Web服务网管接口，简单来说它是一种Web服务器和应用程序间的通信规范。

## uwsgi
uwsgi是一种通信协议，不过跟WSGI分属两种东西，该协议下速度比较快。

## uWSGI
uWSGI是一个Web Server，并且独占uwsgi协议，但是同时支持WSGI协议、HTTP协议等，它的功能是把HTTP协议转化成语言支持的网络协议供python使用。

## uwsgi的使用
在做Django项目时，一般测试开发我们直接用Django内嵌的Web Server即可，但是如果项目要上生产环境，考虑并发等性能时，我们可能需要uwsgi和nginx，下面只说明uwsgi的常用用法，至于nginx的配置笔者后续准备专门写一篇博文来讲。

### 1. 安装
```bash
pip install uwsgi
```

### 2. 配置
uwsgi执行一般有两种方式：命令行和文件配置，但是命令行可能需要识记很多参数，因此采用文件配置是更通用的做法，文件格式支持很多种比如ini、xml、yaml等，笔者建议还是采用比较简单key-value形式ini模式，下面给出一个简单的uwsgi ini配置实例：

```bash
[uwsgi]
socket = 127.0.0.1:8001
master = false
chdir = /var/www/cmpvirtmgr/
module = cmpvirtmgr.wsgi
home = /var/www/env
workers = 2
reload-mercy = 10
vacuum = true
max-requests = 1000
limit-as = 512
buffer-size = 30000
pidfile = /etc/uwsgi/uwsgi.pid
```
执行：uwsgi --ini /path/to/uwsgi.ini

参数解释：
* socket：socket文件，也可以是地址+端口；
* master：是否启动主进程来管理其他进程；
* chdir：项目的根目录；
* module：wsgi文件相对路径；
* home：虚拟环境目录；
* workers：开启的进程数量；
* reload-mercy：设置在平滑的重启（直到接收到的请求处理完才重启）一个工作子进程中，等待这个工作结束的最长秒数；
* vacuum：服务结束后时候删除对应的socket和pid文件；
* max_requests：每个工作进程设置的请求上限；
* limit_as：限制每个uwsgi进程占用的虚拟内存数目；
* buffer_size：设置用于uwsgi包解析的内部缓存区大小；
* pid_file：指定pid文件；
* harakiri：请求的超时时间；
* daemonize：进程后台执行，并保存日志到特定路径；如果uwsgi进程被supervisor管理，不能设置该参数；

更多uwsgi参数可参考官方文档：[https://uwsgi-docs.readthedocs.io/en/latest/](https://uwsgi-docs.readthedocs.io/en/latest/)
