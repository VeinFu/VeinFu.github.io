---
layout: post
title: Openstack云主机实现快照分析
tags:
  - Openstack
comments: true
---

* 目录
{:toc}

## 1. 序言
首先，本文主要是基于代码层面来逐步分析如何创建云主机快照，并且参考的是Opanstack Mitaka/Pike版本的代码，另外默认环境glance、nova和cinder都是采用`ceph`做后端存储。

一般情况下，云主机启动源分镜像启动（`boot from image`）和卷启动（`boot from volume`），而不同的启动源的云主机对应创建虚机快照的实现方式不同，另外，快照本身根据环境又分在线快照和离线快照两种。下面从两种不同启动源的云主机来分别讲述虚机快照具体的实现过程。

另外，Openstack创建的快照跟我们传统意义理解的快照是不一样的，当云主机出现故障时，它没法通过这种快照来进行回滚，不过却可以通过新建的快照来创建新的虚机。

## 2. 镜像启动云主机快照
只对系统盘进行快照，不对其他挂载的磁盘快照；没法对内存及主机状态快照；没有快照链信息，不支持虚拟机回滚至某个快照点；
###离线快照
* 核心代码逻辑可参见：`./nova/virt/libvirt/imagebackend.py`
![](https://upload-images.jianshu.io/upload_images/12911861-68e0f8f7ad7042fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

具体的流程大致解析一下：
* 复制image上传自Glance；
* 基于原先的image创建快照并对image进行clone；
* 执行flatten操作将新的image跟原先的image解耦；
* 对新image创建快照；
* 更新Glance image的location信息；

整个过程中主要开销在flatten操作，该操作需要和源rbd image合并，如果新image中不存在的对象，则需要从源镜像中拷贝，因此IO开销非常大，花费的时间也较长。

### 在线快照
标准快照是通过`qemu-img`从RBD转换而来，目前条件下运行`live_snapshot`不安全，因此采用CEPH为统一后端存储暂不支持这种方式的快照。

### 一些讨论
`为什么 Nova 不实现虚机的快照而只是系统盘的快照呢？据说，社区关于这个功能有过讨论，讨论的结果是不加入这个功能，原因主要有几点：`
* 这应该是一种虚拟化技术的功能，不是云计算平台的功能。
* openstack 由于底层要支持多种虚拟化的技术，某些虚拟化技术实现这种功能比较困难。
* 创建的 VM state snapshot 会面临 cpu feature 不兼容的问题。
* 目前 libvirt 对 QEMU/KVM 虚机的外部快照的支持还不完善，即使更新到最新的 libvirt 版本，造成兼容性比较差。

## 3. 后端卷启动云主机快照
对于后端卷启动的云主机建快照，其主要工作其实是由Cinder来完成的，Cinder对系统盘及其他挂载磁盘快照，同时会上传一个image至Glance，而这个image没什么实质内容，不过image metadata会记录各个磁盘快照的信息。

* 具体的接口实现请参照：`./nova/compute/api.py`
![](https://upload-images.jianshu.io/upload_images/12911861-56e00e69c2f64f22.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)
* 上传至Glance的image的属性信息：
![](https://upload-images.jianshu.io/upload_images/12911861-43be798f9dd8e26d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

最后，如果Openstack统一采用CEPH作为存储后端，建议上传的虚机镜像的格式都是RAW，这样创建虚机或者卷一般都可以实现秒级。

## 4. 延伸

### 传统主流快照的实现
这里是指KVM虚拟机的快照，它可以用来保存某个时间点虚机的磁盘、内存及主机其他的状态，如果该虚机出现故障时，可以利用快照对他进行回滚，快照的实现是通过libvirt来完成。
#### 快照分类
* 根据做快照的对象的不同，快照可分为磁盘快照和内存快照，他们共同构成一个系统还原点，记录某个时间点虚机的完整状态；
* 根据制作快照虚机的活动状态，快照可以分为在线快照和离线快照；
* 根据快照的存储位置，快照又可以分为内部快照和外置快照；

下面将对这几种快照做一个简单的说明：
* 通过`virsh list --all`查看所有创建的虚机
![](https://upload-images.jianshu.io/upload_images/12911861-8777e398ec8c21f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/560)

#### 内存快照
内存快照的实现及恢复比较简单，可参考如下命令：
```bash
# 以虚机fuel-9.0-vienfu_slave-07为例
# 创建内存快照，会保存至一个文件
virsh save fuel-9.0-vienfu_slave-07 slave07-mem.sanpshot
# 内存快照创建完成后，虚机会处于shut off状态
# 回滚快照，注意：只能对处于关机状态的虚机进行回滚
virsh restore fuel-9.0-vienfu_slave-07 slave07-mem.sanpshot
# 完毕后，虚机回到running状态
```
另外虚机内存快照做完后，如果虚机磁盘文件发生修改，可能会造成corruption。

#### 内部快照
磁盘内部快照只支持qcow2文件格式，且快照及后续的变动都会保存在原始的qcow2文件里。`另外，可以对正处于运行状态的虚机做快照，不过快照期间该虚机会处于paused状态，直到快照创建完毕后虚机会重新回到running状态。`
```bash
# 下面以虚机fuel-9.0-vienfu_slave-ctl-02为例
# 创建快照
virsh snapshot-create-as --domain fuel-9.0-vienfu_slave-ctl-02 --name ctl02-disk.snapshot

# 回滚快照
virsh snapshot-revert --domain fuel-9.0-vienfu_slave-ctl-02 --snapshotname ctl02-disk.snapshot
# 删除快照
virsh snapshot-delete --domain fuel-9.0-vienfu_slave-ctl-02 --snapshotname ctl02-disk.snapshot
```
* 如下两张图片会列出新建快照的信息：
![](https://upload-images.jianshu.io/upload_images/12911861-3d78723c78dd69b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/560)
![](https://upload-images.jianshu.io/upload_images/12911861-4f7f2d8ff653af34.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/560)

从第二张图中很明显可以看出，新建的快照会保存到原始的qcow2文件中。同时，从上文中我们不难总结磁盘内部快照存在两个明显的缺点：只支持qcow2格式的镜像文件；快照过程中虚机会转paused状态，这对于线上运行的业务来说不太能接受。

#### 外置快照
外置快照在创建时，快照不会保存在原始的qcow2格式的镜像文件里，而是会单独保存在一个新建的qcow2文件，原始的镜像文件会成为快照的`backing file`（并且受保护，只读），多次创建快照则会存在一个`backing chain`，另外外置快照支持RAW、qcow2格式的虚机镜像文件。

##### 创建外置快照的几种方式
* 只创建磁盘的外置快照
```bash
# 解释一下几个参数
# disk-only：只创建磁盘外置快照
# atomic：存在多个磁盘是，要么全部创建快照成功，要么失败
# no-metadata：不记录快照的元数据
virsh snapshot-create-as --domain <domain_name> --name <snapshot_name> --disk-only --atomic --no-metadata
```
暂不支持快照的回滚，只能去手动实现。
* 创建磁盘和内存快照（不加--live参数）
```bash
virsh snapshot-create-as --domain <domain_name> --name <snapshot_name> --memspec <file_name>,snapshot=external --diskspec <disk_name>,snapshot=external
```
创建快照时需要虚机paused。
* 创建磁盘和内存快照（加--live参数）
```bash
virsh snapshot-create-as --domain <domain_name> --name <snapshot_name> --memspec <file_name>,snapshot=external --diskspec <disk_name>,snapshot=external --live
```
尽管加上`--live`参数，但是创建快照期间虚机还会短暂地paused住，只不过时间短了些较上面一种方式。可以使用 `sanpshot-revert` 命令来回滚到指定的系统还原点，不过得使用 `-force`参数。

## 5. Nova之本地镜像文件做系统盘在线快照

### 背景
当后端使用本地存储时，默认情况下是不支持在线（热）快照的（至少Queens版本之前是默认不支持的），但是实际业务上往往会有自定义镜像（自定义镜像可以这么理解：在虚机中新安装了某些软件或是服务，然后打快照，这样后续我们就可以使用快照镜像来复制新的虚机了）的需求，而如果自定义镜像过程中虚机出现中断的现象，这对于实际运行业务的虚机来讲显然是不合理的也是不能够接受的，所以支持在线快照刻不容缓。

### 配置要求
首先对`libvirt`和`qemu`的版本有最低要求；
修改nova配置文件：
```bash
[workarounds]
disable_libvirt_livesnapshot = false
```
### 代码更新
对于Mitaka版本，在线快照函数`_live_snapshot()`有bug需要更新，具体可参照：https://github.com/openstack/nova/commit/0f4bd241665c287e49f2d30ca79be96298217b7e
