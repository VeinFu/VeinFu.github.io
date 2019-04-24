---
layout: post
title: Python之Mock初探
category: 技术博客
tags:
  - Python
  - 后端开发
comments: true
---

* 目录
{:toc}

##Mock是什么
Mock这个词在英语中是模拟的意思，因此我们可以猜测出这个库的主要功能是模拟一些东西。准确的说，Mock是Python中一个用于支持单元测试的库，它的主要功能是使用mock对象替代掉指定的Python对象，以达到模拟对象的行为。

试想如下应用场景：
假设你开发的项目叫a，里面包含了一个模块b，模块b中的一个函数c（也就是a.b.c）在工作的时候需要调用发送请求给特定的服务器来得到一个JSON返回值，然后根据这个返回值来做处理。如果要为a.b.c函数写一个单元测试，该如何做？

首先想到的简单的方法是搭建一个服务器用来跟用户交互，不过这可能存在各种各样的现实问题，比如搭建服务器的成本（如时间／金钱等）很高，又或者测试服务器无法返回所有可能的值或者需要大量的工作才能达到这个目的。那么此时就可以利用mock替换掉测试服务器与用户的交互过程，这样返回值也会由mock对象来决定，而不需要测试服务器的参与。

##Mock的安装及导入
```python
# python 3.3以前的版本中，需要额外安装mock模块
pip install mock
import mock
# 从python 3.3开始，mock被集成到unittest模块中，可以直接import
from unit test import mock
```
##Mock对象
###基本用法
Mock对象是mock模块中最重要的概念。Mock对象就是mock模块中的一个类的实例，这个类的实例可以用来替换其他的Python对象，来达到模拟的效果。Mock类的定义如下：
```python
class Mock(spec=None, side_effect=None, return_value=DEFAULT, wraps=None, name=None, spec_set=None, **kwargs)
```
mock一个对象的一般步骤如下：
1. 确定替换对象，是一个函数／类／类的实例；
2. 建立mock对象（Mock实例化），并且设置这个对象的行为；
3. 用建立的mock对象代替步骤1确定要替换的对象；
4. 编写UT；

举个栗子：有一个简单的客户端实现，用来访问一个url，当访问正常时，需要返回状态码200，否则返回状态码404，客户端实现代码如下：
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# client.py

import request
def send_request(url):
    r = request.get(url)
    return r.status_code

def visit_google():
    return send_request('www.google.com')
```
外部模块访问谷歌，下面我们使用mock对象在UT中分别测试访问正常和不正常的情况。
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import client, mock, unittest
class TestClient(unittest.TestCase):

    def test_success_request(self):
        success_send = mock.Mock(return_value='200')
        client.send_request = success_request
        self.assertEqual(client.visit_google(), '200')

 def test_fail_request(self):
        fail_send = mock.Mock(return_value='404')
        client.send_request = fail_request
        self.assertEqual(client.visit_google(), '404')
```
###mock用法拓展
####class Mock参数
下面仅列出几个常用重要的参数：
- name:  这个是用来命名一个mock对象，只是起到标识作用，当你print一个mock对象的时候，可以看到它的name。
- return_value: mock对象被调用返回的值，默认值是一个mock对象。
- side_effect: 这个参数指向一个可调用对象，一般就是函数。当mock对象被调用时，如果该函数返回值不是DEFAULT时，那么以该函数的返回值作为mock对象调用的返回值。

其他的参数可参考官方文档: <http://www.voidspace.org.uk/python/mock/mock.html>
####mock对象的自动创建
当访问一个mock对象中不存在的属性时，mock会自动建立一个子mock对象，并且把正在访问的属性指向它，这个功能对于实现多级属性的mock很方便。
####对方法调用进行检查
mock对象有一些方法可以用来检查该对象是否被调用过、被调用时的参数如何、被调用了几次等。实现这些功能可以调用mock对象的方法，具体的可以查看[mock官方文档](http://www.voidspace.org.uk/python/mock/mock.html)。
##patch和patch.object
###patch
####语法
```python
from mock import patch
patch(target, new=DEFAULT, spec=None, create=False, spec_set=None, autospec=None, new_callable=None, **kwargs)
```
具体参数的讲解可参见[patch官方文档](http://www.voidspace.org.uk/python/mock/patch.html#patch)
####运用
一般地patch有三种用法：函数装饰器，类装饰器，上下文管理器，到这初学者可能会一脸懵逼，话不多说直接上代码。

先写一个公共代码文件*common.py*方便上面所说三种用法的调用。
```python
class something(object):
    def do_something():
        return 'Done'
```
测试patch三种用法的代码*test_patch.py*如下:
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import common
from mock import patch

def new_func():
    print('welcome')

# 函数装饰器
@patch('common.something.do_something')
def test_patch(do_sth):
    do_sth.return_value = 'Not Done'
    print(common.something.do_something())

if __name__ == '__main__':
    patcher = patch('common.something', first='one', second='two') #类装饰器
    mock_thing = patcher.start()
    print(mock_thing.first)
    print(mock_thing.second)
    patcher.stop()

    test_patch()

    # contextmanager
    with patch('common.something.do_something') as mock_func: 
        mock_func.side_effect = [1,2,3]
        print(common.something.do_something())
        print(common.something.do_something())
        print(common.something.do_something())

    patcher = patch('__main__.new_func') # 函数装饰器
    patcher.start()
    new_func()
    patcher.stop()
    new_func()

#输出
#one
#two
#Not Done
#1
#2
#3
#welcome
```
###patch.object
####语法定义
```python
#patch the named member (attribute) on an object (target) with a mock object
patch.object(target, attribute, new=DEFAULT, spec=None, create=False, spec_set=None, autospec=None, new_callable=None, **kwargs)
```
####运用
从上面的语法定义不难看出，patch.objec跟patch的用法基本上是一样的，只是使用格式略微不同，举个栗子，使用patch.object来替换上面的test_patch。
```python
# -*- coding: utf-8 -*-

import common
from mock import patch

@patch.object(common.something, 'do_something')
def test_patch(do_sth):
    do_sth.return_value = 'Not Done'
    print(common.something.do_something())
```
总的来说，patch和patch.object通过重新定义函数或类的行为来掩藏原来函数或类的行为，在一些应用场景比如UT测试会有神奇的效果。
