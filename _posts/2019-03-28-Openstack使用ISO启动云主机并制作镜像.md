---
layout: post
title: Openstack使用ISO启动云主机并制作镜像
tags:
  - Openstack
  - 运维实施
comments: true
---

* 目录
{:toc}

## 调研结果
经过M版测试发现openstack其实已经支持这部分功能了，现将具体的操作流程做以下记录。
## 整体操作流程
* 下载ISO镜像并将其上传至glance管理；
* 基于ISO镜像启动一个云主机实例；
* 创建一个空的volume并将其attach到上一步创建的云主机实例；
* 登录到VNC Console完成OS的安装；
* 删除实例，基于volume新建一个云主机实例；
* 登录到新建实例进行必要软件包的安装；
* 信息清除并删除新建实例；
* 基于volume上传image；
* 基于新的image新建云主机实例；

下面仅就一些需要额外注意的步骤做简单说明。
## 上传ISO镜像
以Ubuntu16.04为例，首先[下载ISO](http://ftp.sjtu.edu.cn/ubuntu-cd/16.04/)，然后将该ISO镜像上传至Glance管理。
openstack上传镜像一般可以通过以下两种方式来完成：

* `上传镜像CLI命令`
```bash
openstack image create --disk-format iso --container-format bare --public --file ./ubuntu-16.04.6-server-amd64.iso ubuntu-iso
```
* `openstack Horizon界面操作`
![upload_image.png](https://upload-images.jianshu.io/upload_images/12911861-01c242beff8156e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

`紧接着基于ISO创建云主机不再赘述，直接跳过到下一步。`

## 新建空卷
* 新建一个空的volume，注意卷的大小，至少要大于ISO文件的大小。
![create_blank_volume.png](https://upload-images.jianshu.io/upload_images/12911861-a70c0ca5532c036d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)
然后，把该卷attach到新建的云主机上：
![attach_vol.png](https://upload-images.jianshu.io/upload_images/12911861-459c98af8f074625.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

## 登陆VNC安装OS
* 这一步其实跟我们日常直接用ISO通过制作USB启动介质安装OS或者通过光驱安装OS的过程基本上都是一样的，不过特别注意的也是这一步，因为我们需要特别把OS安装到刚才新建的空的volume里。

* 登陆VNC Console进入安装OS界面：
![os_install.png](https://upload-images.jianshu.io/upload_images/12911861-017498ef52197ebf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

如上所提，操作跟正常安装OS一样，再提醒一遍：当OS安装到指定目标盘或分区时，一定要对应到空的新建的volume。当OS完成安装时，此时其实OS已经安装到我们新建的volume里了，而这时的云主机已经没什么用了可直接删掉，紧接着把volume设置成bootable：
```bash
# 获取volume的ID
cinder list | grep ubuntu16.04_os
# 设置可启动
cinder set-bootable  <volume_id> true
```

然后，基于该volume再新建一个云主机，登陆到云主机后安装一些通用的软件比如`cloud-init、qemu-guest-agent、openssh-server`等，之后删除`/etc/udev/rules.d/70-persistent-net.rules`清除网卡网络配置脚本文件MAC、UUID等信息(或者也可以通过virt-sysprep来清理)，最后删除云主机。

## 基于卷上传镜像
* 上传镜像
![image_upload.png](https://upload-images.jianshu.io/upload_images/12911861-47fb0c4ee6177209.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

* 镜像列表
![image_list.png](https://upload-images.jianshu.io/upload_images/12911861-4c144338ce6811da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

* 然后就可以基于image创建云主机了：
![vms_list.png](https://upload-images.jianshu.io/upload_images/12911861-71d25569b073055f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

* ssh（通过key pair的方式）登陆到云主机：
![login_vm.png](https://upload-images.jianshu.io/upload_images/12911861-31f168c73c6efa14.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)
登陆ok。

`综上所述都是针对于Linux ISO镜像，那对于Windows ISO在线安装及制作的流程是怎样的，对比Linux又有什么需要特别留意的地方呢？`
## 使用Windows ISO启动云主机并制作镜像
对比使用Linux ISO启动云主机，Windows ISO启动云主机的流程跟它基本上是一致的，但是因为Windows ISO不自带虚拟硬盘、网卡的驱动virtio（Linux自带有virtio），所以需要提前下载[virtio ISO]([https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/index.html](https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/index.html)
)，之后在安装Windows OS的过程中导入该驱动进而识别到目标盘。

那究竟通过什么途径在安装Windows OS能够导入virtio驱动呢？如下提供三种方式供参考：
* 第一种方法，参考Linux，通过软件工具（比如UltraISO等）把virtio驱动导入到Windows ISO中后对Windows ISO进行重建，具体构建过程在此不作说明，感兴趣的童鞋可以网上自行搜索。

    `下面两种方法都是通过Openstack来实现的，首先都需上传virtio ISO至Glance（其实就是一般上传虚机镜像的方式来完成）`
* 第二种方法，基于Glance下virtio镜像新建一个卷，之后将该卷attach到云主机上，此时其实还是没法导入virtio驱动的（因为默认新建卷的类型是disk，且bus为virtio，所以也就识别不到新挂载的卷），不过可以通过手动修改云主机的XML文件之后重启云主机来达到目的，修改部分有两块，可参照下图：
![云主机XML.png](https://upload-images.jianshu.io/upload_images/12911861-a045b14778a49629.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

  上图该区域对应的是基于virtio iso所建的卷的配置；
  上图中标记1修改成cdrom；
  上图中标记2修改成如下配置：
  ```bash
  <target dev='hdb' bus='ide'/>
  <readonly/>
  <address type='drive' controller='0' bus='0' target='0' unit='1'/>
  ```
* 第三种方法，借助Openstack启动实例的CLI命令，设置`block-device`参数，具体样式可参考如下：
  ```bash
  # 其中image参数指代的是Windows ISO镜像的ID，而block-device参数的id则指代的是virtio镜像的ID
  nova boot --image b4ba82ca-beaa-4266-81a4-9ff23ec9d524 --flavor 2 --nic net-id=1af38e89-0d44-4508-b5af-c77ea173667d --block-device source=image,dest=volume,id=acddaec0-a2db-4cae-ab05-327443cf15fe,type=cdrom,bus=ide,size=1 mytest
  ```

综合比较三种方法，建议采用`第三种方式`，不过目前这种方式只支持CLI形式无法直接在Openstack Dashboard上操作完成，至于其他操作流程，还请参照Linux ISO启动云主机操作流程。
