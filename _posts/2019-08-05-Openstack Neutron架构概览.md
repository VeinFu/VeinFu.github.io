---
layout: post
title: Openstack Neutron架构概览
category: 技术博客
tags:
  - Python
  - Openstack
comments: true
---

* 目录
{:toc}

## Neutron是什么

Neutron跟Nova、Cinder等一样都是Openstack的核心服务组件，它主要负责管理和维护Openstack网络资源。Neutron大体的架构图如下：
* Neutron架构图
![Neutron架构图](https://upload-images.jianshu.io/upload_images/12911861-848915d43d6488f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/560)

从Neutron的架构图不难看出Neutron的大体服务构成以及各个服务组件的职能，而它一般的工作流程如下：
* Neutron Server接收用户请求，把请求存放到消息队列中；
* Neutron Plugin从MQ中获得请求，在DB中维护网络的状态信息，然后把请求存放到MQ等待Neutron Agent处理；
* Neutron Agent从MQ中获取Plugin的请求，交由Network Provider处理具体的网络请求；

架构图上没有提到的Network Provider，其实它是提供网络服务的虚拟或物理网络设备，例如linux bridge、Open VSwitch或其他Neutron支持网络交换机，实际使用中Neutron配置中采用什么样比如linux bridge的provider，对应的plugin和agent就要采用linux bridge的plugin和agent。

## Neutron支持网络类型

Neutron目前支持的网络类型一共有五种：local、flat、vlan、vxlan和gre五种网络，其中gre网络目前在linux bridge还不支持。
* local：不会绑定到实际的网卡，只有同一个节点下的同一个网络下的虚机才能通信，不能跨节点；
* flat：没有tag的网络，位于同一个网络下的虚机可以通信，且可以跨节点；
* vlan：是Neutron最常用的网络类型，同一个vlan的虚机可以通信可以跨节点，不同的vlan通过tag在二层上是网络隔离的，不过在三层可以通过router使之相互通信；
* vxlan和gre都是基于tunnel技术的overlay网络，只不过前者封装的是UDP包而后者封装的则是IP包；

## Neutron几个重要概念

### network
上面说的网络类型指的就是这个，其实它是一个二层的概念，不同的network在二层是隔离的。

### subnet
是一个IPV4或IPV6地址范围，具体的虚机会通过它来分配IP，每一个subnet都应包括两个部分：ip地址范围和掩码，比如192.168.0.0/24。

### port
是一个逻辑的概念，可以理解成虚拟网口，虚机需要跟它绑定，同样路由也需要跟它绑定，有mac和ip，要关联具体的network和subnet。

### router
实现不同网段间的互相通信，分为物理路由和虚拟路由。
