---
layout: post
title: Supervisor学习记录
category: 技术博客
tags:
  - 运维实施
comments: true
---

* 目录
{:toc}

## 方法论

学习supervisor之前首先得搞明白什么是supervisor以及它的功能是什么。supervisor是一款用python开发的进程管理工具，由它管理的进程或者程序不能是daemon进程，因为supervisor会自动把它们装成daemon进程的，所以比如要用supervisor来管理nginx需要设置`daemon off`。supervisor目前在Web应用中非常广泛，一个Web网站常见的部署套路：django+uwsgi+nginx+supervisor。

## supervisor的兼容问题
当前supervisor只支持python2.x，所以安装supervisor的时候，先把系统默认的python版本调成python2.x；如果需要管理python3的程序，把启动程序的python编辑器改成python3即可。

## supervisor配置
一般地如果在Ubuntu OS下是通过`apt-get install supervisor`安装，supervisor会默认开机启动且在`/etc/supervisor`目录自动生成supervisord.conf；如果是其他OS可能会有出入，我们可以参照一般Linux下设置开机启动的方式来设置supervisor开机启动，而配置文件模板可参照如下方法去生成：

#### 生成supervisor配置模版
```python
echo_supervisord_conf > /etc/supervisor/supervisord.conf
```
#### 配置supervisord.conf
supervisor的配置文件一般可分两大块：通用部分和APP部分，基本上通用部分我们一般使用其默认的配置即可，而对应每个被管理的进程的配置可以放在`/etc/supervisor/conf.d`目录下，来通过在supervisord.conf设置`include conf.d/*.conf`

下面是一个简单的通过supervisor来管理celery的队列任务、定时任务及监控进程配置的样板：
```python
[program:celery_worker]
autorestart = true
autostart = true
stopasgroup = true
command = /home/zaihui/server/env/bin/celery worker --app=core --loglevel=WARN --concurrency=4 --events --queue celery,hi_priority --pidfile=/var/run/celery_worker.pid
directory = /home/zaihui/server/
numprocs = 1
startsecs = 10
stderr_logfile = /var/log/supervisor/celery_error-%(process_num)s.log
stdout_logfile = /var/log/supervisor/celery-%(process_num)s.log
environment =
    AWS_PROD=True,
    PYTHONPATH=/home/zaihui/server/ygg,
    C_FORCE_ROOT=True,
    AWS_ACCESS_KEY_ID="AKIAOIA6TYIGJ2GE7WJQ",
    AWS_SECRET_ACCESS_KEY="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"

[program:celery_beat]
autorestart = true
autostart = true
stopasgroup = true
directory = /home/zaihui/server/
command = /home/zaihui/server/env/bin/celery beat --app=core --loglevel=WARN --pidfile=/var/run/celery_beat.pid
numprocs = 1
startsecs = 10
stdout_logfile = /var/log/supervisor/celery_beat-%(process_num)s.log
stderr_logfile = /var/log/supervisor/celery_beat_error-%(process_num)s.log
environment =
    AWS_PROD=True,
    PYTHONPATH=/home/zaihui/server/ygg,
    C_FORCE_ROOT=True,
    AWS_ACCESS_KEY_ID="AKIAOIA6TYIGJ2GE7WJQ",
    AWS_SECRET_ACCESS_KEY="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"

[program:celery_flower]
autorestart = true
autostart = true
stopasgroup = true
directory = /home/zaihui/server/
command = /home/zaihui/server/env/bin/celery flower --app=core --conf=/etc/supervisor/conf.d/flowerconfig.py --broker_api=http://zaihui:rejn18cyly@54.223.38.157:15672/api/
numprocs = 1
startsecs = 10
stdout_logfile = /var/log/supervisor/celery_flower-%(process_num)s.log
stderr_logfile = /var/log/supervisor/celery_flower_error-%(process_num)s.log
environment =
    AWS_PROD=True,
    PYTHONPATH=/home/zaihui/server/ygg,
    C_FORCE_ROOT=True,
    AWS_ACCESS_KEY_ID="AKIAOIA6TYIGJ2GE7WJQ",
    AWS_SECRET_ACCESS_KEY="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"
```
`不过一般对于被supervisor管理的不同程序或进程，我们都会在/etc/supervisor/conf.d编辑一个配置文件，然后再通过/etc/supervisor/supervisord.conf文件include conf.d/*.conf`

#### 启动或关闭supervisor服务
supervisord -c /etc/supervisor/supervisord.conf  # 启动
supervisord -c /etc/supervisor/supervisord.conf shutdown  # 关闭
supervisord -c /etc/supervisor/supervisord.conf reload  # 重启
supervisorctl stop all  # 关闭所有进程
supervisorctl stop <app_name>  # 单独关闭某个进程

#### docker中supervisord的使用
注意，如果应用程序放在docker镜像里通过容器启动的话，docker容器的启动命令应该是`supervisord -n -c /etc/supervisor/supervisord.conf`