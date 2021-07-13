---
author: admin
comments: true
date: 2011-08-26 03:16:18+00:00
layout: post
slug: webpy_middleware
title: web.py的中间件和钩子
wordpress_id: 43
categories:
- web
tags:
- hook
- middleware
- web.py
---

web.py有添加中间件和钩子的功能

钩子在web.py的cookbook中有介绍,
中间件可以理解为app的钩子,
其作用范围可以大致通过web.ctx是否初始化看出来.

中间件是一个callable的对象,初始化参数是app,
调用参数就是app的调用参数

下面的代码说明了两种创建中间件的方式(def定义和class定义),
以及中间件和钩子的执行顺序:

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import web

def midd(app):
    """函数规则定义的中间件"""
    def inapp(e, o):
        print 'func befor handle'
        r = app(e, o)
        print 'func after handle'
        return r
    return inapp

class Midd(object):
    """class规则定义的中间件,
    其实这就是一个函数,
    传入参数时调用__init__,
    然后返回self"""
    def __init__(self, app):
        self.app = app
    def __call__(self, o, e):
        print 'class befor handle'
        print web.ctx
        r = self.app(o, e)
        print 'class after handle'
        return r

def my_processor(handler):
    print 'hook before handling'
    print web.ctx
    result = handler()
    print 'hook after handling'
    return result
urls = ("/.*", "hello")
app = web.application(urls, globals())
app.add_processor(my_processor)

class hello:
    def GET(self):
        print 'start handle'
        return 'Hello, world!n'

if '__main__' == __name__:
    app.run(Midd)    #使用class中间件
    #app.run(midd)   #使用函数中间件
```

执行和输出信息

```python
$ python index.py
http://0.0.0.0:8080/
class befor handle
<ThreadedDict {}>
hook before handling
<ThreadedDict {'status': '200 OK', ......}>
start handle
hook after handling
127.0.0.1:35260 - - [26/Aug/2011 11:01:59] "HTTP/1.1 GET /" - 200 OK
class after handle
```
