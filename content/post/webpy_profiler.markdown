---
author: admin
comments: true
date: 2011-08-27 04:51:09+00:00
layout: post
slug: webpy_profiler
title: web.py中自己编写的请求处理的性能分析
wordpress_id: 57
categories:
- web
tags:
- profile
- web.py
---
web.py本身提供profile中间件 `web.profiler`

在wsgi和自带的httpserver中同样适用

只需要在`app.run`或者`app.wsgifun`中传入传入中间件参数`web.profiler`就可以了

在每个请求返回的网页内容之后会打印分析数值但是有时我们需要数值在后台显示,而不是显示在页面中,这时候我们可以自己改写一下profile中中间件

```python
class Profile_Middleware(object):
    def __init__(self, app):
        self.app = app
    def __call__(self, mapping, fvars):
        cp = cProfile.Profile()
        r = cp.runcall(self.app, mapping, fvars)
        cp.print_stats()
        return r
```

把Profile_Middleware当中间件传给web.py,这时候就会正常返回显示页面,
分析信息打印在命令行下了,如果需要打印日志,可以自行更改`__call__`函数
