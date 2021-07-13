---
author: admin
comments: true
date: 2011-09-23 10:29:17+00:00
layout: post
slug: decorators-in-webpy
title: 在web.py中使用装饰器
wordpress_id: 135
categories:
- web
tags:
- decorators
- python
- web.py
---

先上代码

```python
import web
urls = (
    '/', 'hello',
)
app = web.application(urls, globals())
def hold(func):
    print 'run hlod'
    def f(self):
        print 'in hold func'
        return 'hold'
    return f
class hello:
    @hold
    def GET(self):
        print 'in hello'
        return 'hello'

if __name__ == "__main__":
    app.run()
```

现在的问题是,hold会在什么时候调用?

过去我一直以为是每次调用GET函数的时候会调用一遍hold函数,
但事实并不是这样的.

事实是,当定义GET函数加载的时候,就会调用一遍hold函数,
而且仅仅调用一遍hold.

这时候hold返回的函数就会替代原始的GET函数定义,
所以运行这点python代码,输出结果如下:

```shell
http://0.0.0.0:8080/
run hlod
in hold func
10.0.2.2:51602 - - [23/Sep/2011 08:04:11] "HTTP/1.1 GET /" - 200 OK
10.0.2.2:51602 - - [23/Sep/2011 08:04:11] "HTTP/1.1 GET /favicon.ico" - 404 Not Found
in hold func
10.0.2.2:51602 - - [23/Sep/2011 08:04:12] "HTTP/1.1 GET /" - 200 OK
```
