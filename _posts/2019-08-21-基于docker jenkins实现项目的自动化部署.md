---
layout: post
title: 基于docker jenkins实现项目的自动化部署 
category: 技术博客
tags:
  - docker
  - 运维实施
comments: true
---

* 目录
{:toc}

## 1. jenkins安装
如文档题目，此处jenkins是基于docker来安装部署的，那么首先需要先安装docker，[这里](https://www.jianshu.com/p/911dba395ef3)有docker具体的安装教程，此处不再赘述。

docker安装完毕，接着开始jenkins的安装及部署：
### 1.1 拉取jenkins镜像
这一部分很简单，一个命令就可以完成：
```bash
docker pull jenkins/jenkins:latest
```
### 1.2 订制jenkins镜像
实际使用场景中，我们可能需要额外安装一些软件工具，比如fabric(一种基于ssh的轻量自动化部署工具)；另外直接拉取的镜像默认用户是jenkins，存在文件的访问权限的问题，通过重新编辑Dockerfile可以把jenkins的用户改成root，下面是一个简单的Dockerfile的样例：
```bash
FROM jenkins/jenkins:latest
USER root

RUN apt-get update \
    && apt-get install -y python-pip \
    && pip install fabric3
ARG dockerGid=998
RUN echo "docker:x:${dockerGid}:jenkins" >> /etc/group
```
然后，运行命令重新构建jenkins镜像：
```bash
docker build -t vienfu_jenkins -f Dockerfile .
```
### 1.3 启动容器
jenkins镜像订制完成后，下面就是启动jenkins容器：
```bash
docker run -d --name my_jenkins --restart=always -p 8006:8080 \
-v /root/jenkins_home:/var/jenkins_home \  # 挂载jenkins工作目录，方便备份和迁移
-v /var/run/docker.sock:/var/run/docker.sock \ # 包括以下两项的挂载，可以实现在jenkins容器内部构建docker并启动docker容器的目的
-v /usr/bin/docker:/usr/bin/docker \
-v /usr/lib/x86_64-linux-gnu/libltdl.so.7:/usr/lib/x86_64-linux-gnu/libltdl.so.7 \
vienfu_jenkins
```
另外，如果jenkins容器要跟宿主机实现ssh免密访问，可额外添加如下操作：
```bash
docker exec my_jenkins ssh_keygen -t rsa  # 一路回车即可
docker exec my_jenkins cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys
```
## 2. 新建jenkins任务
jenkins容器启动之后，在浏览器输入：`<宿主机IP>:8006`即可打开jenkins的UI，首次打开会提示输入管理员密码，可在宿主机输入以下命令把结果粘过去即可：
```bash
docker exec my_jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
接着安装jenkins插件，选择常规安装即可，也可以跳过手动选择安装。

然后我们就看到了jenkins的主页面了，选择`系统管理 -- 管理用户`新建jenkins用户并完成登录。
* 系统管理入口
![](https://upload-images.jianshu.io/upload_images/12911861-7814a9ca1eaca874.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/560)
* 新建用户页面
![](https://upload-images.jianshu.io/upload_images/12911861-046341b463f0f02e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/560)

接着安装必要插件，回到jenkins主页面，选择`系统管理 -- 插件管理`来安装所需要插件，比如gitlab plugin、git plugin、ssh plugin等。
* 插件安装入口
![](https://upload-images.jianshu.io/upload_images/12911861-5aa26870b0c5467f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/560)

最后，新建任务，回到jenkins主页面，选择`新建任务 -- 输入项目名称 -- 选择自由风格的软件项目`完成任务的创建；点开新建的任务做一些任务的配置：
* 任务配置入口
![](https://upload-images.jianshu.io/upload_images/12911861-f63ac14a934639b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/560)
* 配置源码管理，选择`git`
![](https://upload-images.jianshu.io/upload_images/12911861-677c8ef2ddb97444.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/560)
* 构建选择`执行shell`，编辑自动化部署的脚本，可以是纯shell脚本也可以是诸如fabric、ansible这样自动化的工具来实现
![](https://upload-images.jianshu.io/upload_images/12911861-a82e6a62ba3d0e18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/560)
上图是通过shell+fabric来实现一个docker镜像的build、启动和导出，既然是fabric那就需要一个`fabfile.py`，它的内容如下：
  ```bash
  # -*- coding: utf-8 -*-

  from fabric.api import *

  env.roledefs={
      'jenkins': ['root@172.17.0.1']
  }

  @task
  @roles('jenkins')
  def build_and_import_docker_img():
      print('Just a jenkins demo test.')
      local('docker build -t cmpvirtmgr -f deploy/deploy_env/Dockerfile .')
      with cd('/root'):
          run('docker run -d --network=host --restart=always cmpvirtmgr')
          run('docker save -o cmpvirtmgr.tar cmpvirtmgr:latest')
  ```

如此就基本就完成一个简单项目的自动化部署。

## 3. 题外篇
可能是出于安全的考虑，一般我们在新订制jenkins镜像最后还是还原设置`USER jenkins`，这样如果要在宿主机通过fabric免密部署项目，如下两步需要操作：
* jenkins容器生成jenkins key pair
```bash
docker exec my_jenkins ssh-keygen -t rsa # 一路回车即可，生成的key会默认保存在/var/jenkins_home/.ssh/目录下
```
* 新建jenkins用户
因为jenkins的容器默认的用户是jenkins，所以需要在jenkins宿主机新建jenkins用户并且要实现从jenkins容器ssh key免密登陆，具体操作可参照如下：
```bash
# 以下操作均在jenkins宿主机上操作
# 1. 添加jenkins用户
useradd jenkins -d /home/jenkins -m -s /bin/bash
# 2. 设密码
passwd jenkins
# 3. 设定sudo权限
# 编辑/etc/sudoers，默认该文件没有写权限，需要先修改权限，然后往里添加一行：jenkins ALL=(ALL:ALL) ALL
# 4. 切换jenkins用户
su - jenkins
# 5. 在/home/jenkins目录下新建authorized_keys文件，然后把jenkins容器中jenkins用户的公匙添加到这个文件里，并设置权限
chmod 600 authorized_keys
```