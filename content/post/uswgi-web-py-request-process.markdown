---
author: admin
comments: true
date: 2011-09-07 03:48:40+00:00
layout: post
slug: uswgi-web-py-request-process
title: uwsgi部署的web.py一次request调用过程
wordpress_id: 125
categories:
- web
tags:
- uwsgi
- web.py
- python
---

##首先看一下uwsgi对于一次普通wsgi函数的调用过程

uwsgi可以通过一下命令启动

`uwsgi -s :9090 -w index`

其中

+ -s表示监听的端口
+ -w表示python启动模块

例如`index.py`的模块名就是`index`

更多uwsgi的使用可以查看官方文档[http://projects.unbit.it/uwsgi/wiki/Doc](http://projects.unbit.it/uwsgi/wiki/Doc)

uwsgi要求在启动模块中有一个`application`变量
这个变量其实就是一个遵循wsgi标准的处理函数,接受两个参数,例如

```python
#File : index.py
def simple_app(environ, start_response):
    """遵循wsgi标准的函数,
    environ是wsgi传递的http请求参数
    start_response是一个函数,调是需要传递两个参数
        status 状态码 "200 OK" 或者 "404 NOT FOUND"等
        head   http的head头信息
"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return ['Hello world!n']
application = simple_app
```

这时候运行

```bash
uwsgi -s :9090 -w index
```

就会运行在9090端口监听的wsgi程序,
配合nginx进行测试,修改nginx配置文件:

```bash
server{
    listen 80;
    server_name localhost;
    location / {
        uwsgi_pass 127.0.0.1:9090;
        include uwsgi_params;
    }
}
```

重启nginx,访问本机就会出现index.py提供的hello word页面了

##接下来查看web.py的调用过程

修改index.py文件

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import web

urls = (
    '/.*', 'index',
)

app = web.application(urls, globals())

class index(object):
    def GET(self):
        return 'hello web.py'
#除了web.py的基础代码,只是把application改成了app.wsgifunc()
application = app.wsgifunc()
```

去看看wsgifunc()的定义

```python
#web.py的application.py文件的一部分
   def wsgifunc(self, *middleware):
       """Returns a WSGI-compatible function for this application."""
       def peep(iterator):
           #.....
           return itertools.chain([firstchunk], iterator)

       def is_generator(x): return x and hasattr(x, 'next')

       def wsgi(env, start_resp):
           #web.py自己的wsgi处理函数
           #实际上每次处理request就是从这里开始调用的
           # clear threadlocal to avoid inteference of previous requests
           self._cleanup()

           self.load(env)
           try:
               # allow uppercase methods only
               if web.ctx.method.upper() != web.ctx.method:
                   raise web.nomethod()

               result = self.handle_with_processors()
               if is_generator(result):
                   result = peep(result)
               else:
                   result = [result]
           except web.HTTPError, e:
               result = [e.data]

           result = web.safestr(iter(result))

           status, headers = web.ctx.status, web.ctx.headers
           #调用wsgi的状态和head设置函数
           start_resp(status, headers)

           def cleanup():
               self._cleanup()
               yield '' # force this function to be a generator
           #返回wsgi要求的可迭代对象
           return itertools.chain(result, cleanup())
       #裹上我们自己设置的n层中间件,并加入web.py的wsgi处理函数
       #包裹的过程只执行一次
       for m in middleware:
           wsgi = m(wsgi)
       #uwsgi保存的就是这个包裹了中间件的wsgi函数
       return wsgi
```

这里对wsgifunc的执行过程做了注释,每次request的web.py的内部处理过程实际上
就是wsgi这个函数,而wsgifunc的作用是:生成挂载于uwsgi上的wsgi函数.
