---
layout: post
title: Python之Celery学习札记
category: 技术博客
tags:
  - Python
  - 后端开发
comments: true
---

* 目录
{:toc}

##Celery是什么
Celery是一个管理分布式队列的工具，它封装好了常见任务队列的各种操作，能够快速进行任务队列的使用和管理，并且专注于任务处理，支持任务调度，并且特别适合异步任务、耗时任务以及定时任务的处理。

##快速入门

###几个概念理解
想让Celery运行起来首先得理解如下几个概念：
- **Broker**
中文意思是中间人，也可以称为代理，在这儿是指任务队列本身，而Celery则扮演着生产者和消费者的角色，broker是生产者和消费者存储／处理数据的地方。常见的broker有RabbitMQ，Redis，ZeroMQ等。
- **worker**
顾名思义，是Celery的工作者，类似于生产者／消费者中的消费者，它将从任务队列中取出任务并执行。
- **Result Stores／Backend**
结果存储的地方，队列中的任务运行完之后的结果或者状态需要被任务发送者知道，那么就需要一个地方来存储这些结果。
常见的backend有redis，memcached甚至常用的数据库都可以。
- **Tasks**
就是要在队列中进行的任务了，一般是由用户／触发器活着其他操作将任务入队，然后交给worker进行处理。

###举个栗子
这里用redis来作broker和backend，先写一个task。
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# tasks.py

from celery import Celery

app = Celery('tasks', backend='redis://localhost:6379/0', broker='redis://localhost:6379/0')

@app.task
def add(x, y):
    return x + y
```
到这里，有了broker，backend和task，那就可以通过如下命令起一个worker来处理task。
```
celery worker -A tasks --loglevel=info
```
这时候会发现worker并没有实际处理任务，这是因为还没有通过触发器把任务添加到broker里，下面写个触发脚本。
```python
#-*- coding: utf-8 -*-
# trigger.py
from tasks import add
import time

result = add.delay(3, 5)
while not result.ready():
    time.sleep(1)
print('task done: {}'.format(result.get()))
```
运行此脚本，delay 返回的是一个 AsyncResult 对象，里面存的就是一个异步的结果，当任务完成时`result.ready()` 为 true，然后用 `result.get()` 取结果即可。到此，一个简单的 celery 应用就完成啦。

##Celery使用进阶
经过快速入门的学习后，我们可以用Celery对简单的任务进行管理了，但是运用到实际场景，这肯定是远远不够的，所以需要我们去学习更为复杂或者高级的使用方式了。下面介绍几种使用方式：

###1. 绑定任务为实例方法
跟类方法处理的方式一样，需要一个关键字`self`。一旦把任务绑定为实例的方法后，我们就可以在进行任务的时候来查看`Celery.task`的各种属性信息，从而根据结果做很多其他操作，比如判断链式任务是否到结尾等等，举个栗子：
```
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# tasks.py

from celery import Celery
from celery.utils.log import get_task_logger

app = Celery('tasks', backend='redis://localhost:6379/0', broker='redis://localhost:6379/0')

logger = get_task_logger(__name__)
@app.task(bind=True)
def add(self, x, y)
    logger.info(self.request.__dict__)
    return x + y
```

###2. Task继承，重写方法
不啰嗦，直接上示例代码。
```
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# tasks.py

import celery

class mytask(celery.Task):
    def on_failure(self, retval, task_id, args, kwargs):
        print('{} task failed: {}'.format(task_id, retval))

app = celery.Celery('tasks', backend='redis://localhost:6379/0', broker='redis://localhost:6379/0')

@app.task(base=mytask)
def add(x, y)
    raise KeyError()
```

###3. 任务状态回调(callback)
对于耗时任务，实际场景中我们想知道任务的执行状态，可以通过设置`on_message`回调来实现。

`tasks.py`
```bash
!/usr/bin/env python
# -*- coding: utf-8 -*-

import time
from celery import Celery

app = Celery('my_app', broker='redis://localhost:6379/0', backend='redis://localhost:6379/0')

@app.task(bind=True)
def test_status_callback(self):
        for i in xrange(1, 101):
                self.update_state(state='PROGRESS', meta={'bar': i})
                print i
                time.sleep(1)
        return  'Done!'
```
`trigger.py`
```bash
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import sys
import time
from tasks import test_status_callback

def progress_detail(body):
        res = body.get('result')
        if body.get('status') == 'PROGRESS':
                sys.stdout.write('\r任务进度: %s%%' % res.get('bar'))
                sys.stdout.flush()
        else:
                print '\r'
                print res

res = test_status_callback.delay()
print '11111111111'
print '22222222222'
print res.get(on_message=progress_detail, propagate=False)
```
这样启动worker和执行触发脚本之后我们就可以在输出信息上看到实时的任务进度更新。

###4. 定时周期任务(celery-beat)
一般可以通过两种方式来配置：
1. 直接定义beat_schedule，然后添加配置 — app.conf.update(beat_schedule=beat_schedule)
2. 把任务调度的配置写在一个文件，假设文件名`celery-beat.py`，然后在写任务程序的时候调用这个文件 — app.config_from_object(‘celery-beat’)

### 配置的格式，举个栗子：
```python
beat_schedule = {
	‘servant_create_dishes’:  # 任务名，可随意写
		{
			‘task’: <该任务被定义的地方>,
			'schedule': <时间配置>,
		}
}
```
3. 启动worker
```bash
celery -A app_name beat
```

其中，除了task, schedule等常见的field外，还有args，kwargs(其实就是task的位置及关键字参数)；而schedule一般我们可以通过timedelta，crontab等来实现。

###5. 任务路由及优先级的处理
1. 首先定义queue：
```python
_task_queues = (
	Queue('celery', routing_key='celery'),
	Queue('heavy_db', routing_key='heavy_db'),
	Queue('hi_priority', routing_key='hi_priority'),
)
app.conf.update(task_queues=_task_queues)
```
2. 然后定义routes用来决定不同的任务去哪一个queue
```python
_task_routes = {
	'servant.commands.servant_create_dishes': {
		'queue': 'hi_priority',
		'routing-key': 'hi_priority',
	},
}
app.conf.task_routes(task_routes=_task_route)
```
3. 启动worker
```bash
celery -A app_name worker -Q celery,heavy_db,hi_priority
```

###6. 监控
celery的监控可以考虑flower，flower是一个基于web的监控页面，安装及配置都非常简单：
```bash
pip install flower
celery -A my_proj flower --broker redis://localhost:6379/0 # 默认端口5555，如果更改可使用-p参数
```
然后在浏览器输入：`127.0.0.1:5555`即可打开flower监控页面。

###7. 其它
实际做Web项目时，我们经常会遇到一些耗时的任务比如上传/下载大文件、发送消息邮件等，如果前端页面点击类似的上传/发送按钮后一直处于等待状态，这样使用的体验会非常差，此时可以采用celery做异步处理，具体过程如下(仅供思路参考)：
* 前端调用耗时任务接口时，后端立即给前端响应celery任务的id；
* 前端做一个轮询，根据获取的任务id不断地从redis获取任务的执行状态；
* 当获取任务状态为success时进行前端页面渲染显示处理成功；



