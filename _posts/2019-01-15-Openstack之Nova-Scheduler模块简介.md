---
layout: post
title: Openstack之Nova-Scheduler模块简介
category: 技术博客
tags:
  - Openstack
comments: true
---

* 目录
{:toc}

## 1. 楔子
从之前[openstack Nova模块概览](https://www.jianshu.com/p/4d0e71b952ba)这篇文章我们可以大致知道`nova-scheduler`模块的功能：在创建（或者重建）虚机时，无非是`nova-api`接收到HTTP请求后，由`nova-schduler`决定该请求在哪个节点具体执行。而本篇主要是基于应用及代码层面进一步学习`nova-scheduler`的。
## 2. 调度步骤
问题来了，`nova-scheduler`究竟是通过什么方式或算法来选择合适的节点来创建虚机呢？简单来说，基本上就是如下两种途径了：
- Filters：通过各种Filter筛选掉不满足要求的的主机；
- 计算权重：基于上面过滤后的结果对每个主机计算权重值，然后排序，最终选择权重值高的几个主机。
## 3. Filters
### 过滤器配置
可以通过查看控制节点`/etc/nova/nova.conf`来获取，而其中需要留意的是下面三项配置信息：
*`scheduler-driver`*: 调度驱动
![scheduler_driver.png](https://upload-images.jianshu.io/upload_images/12911861-1c5c2fd276148348.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
*`scheduler_available_filters`*:可用的过滤器，默认所有都可用
![scheduler_available_filters.png](https://upload-images.jianshu.io/upload_images/12911861-f9b2a566d868a7d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
*`scheduler_default_filters`*:实际用的过滤器
![image.png](https://upload-images.jianshu.io/upload_images/12911861-36214769e9666c16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 过滤调度实现
下面具体从代码层面一步一步分析过滤调度是如何实现的：
`/nova/conductor/manager.py`
![](https://upload-images.jianshu.io/upload_images/12911861-94d3588b668bb677.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`/nova/scheduler/client/__init__.py`
![](https://upload-images.jianshu.io/upload_images/12911861-e19c349f7735b4ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`/nova/scheduler/client/query.py`
![](https://upload-images.jianshu.io/upload_images/12911861-3077c6d338d11fa7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`/nova/scheduler/rpcapi.py`
![](https://upload-images.jianshu.io/upload_images/12911861-5d4ace56436f4e6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`/nova/scheduler/manager.py`
![](https://upload-images.jianshu.io/upload_images/12911861-f4af07e42b813bfa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`/nova/scheduler/filter_scheduler.py`
![](https://upload-images.jianshu.io/upload_images/12911861-765920987bfda31c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`/nova/scheduler/host_manager.py`
![](https://upload-images.jianshu.io/upload_images/12911861-f5e21c01820db656.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
而函数`get_filtered_hosts`最终实现过滤调度，深挖你会发现这个函数会根据配置文件`scheduler_default_filters`依次调用各个过滤器模块的`host_passes`函数实现对主机的筛选，这儿就不细细赘述了。

另外注意，对于CPU、HDD、Memory等模块是允许`overcommit`，具体的比例值参照配置文件。

## 4. 权重计算
`/nova/scheduler/filter_scheduler.py`这个文件除了实现Filters筛选出符合要求的HOST外，而且也同时实现对这些HOST的权重计算。
![](https://upload-images.jianshu.io/upload_images/12911861-99d995c096bab945.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`/nova/scheduler/host_manager.py`
![](https://upload-images.jianshu.io/upload_images/12911861-eefa4a73df677a03.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
函数`get_weighted_objects`依据不同的Weighter来计算每个HOST的权重值，而对于每个Weighter只需要实现函数`_weight_object`即可，另外每种Weighter的`multiplier`值通过配置文件获取。
