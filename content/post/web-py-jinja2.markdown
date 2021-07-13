---
author: admin
comments: true
date: 2011-09-06 07:37:16+00:00
layout: post
slug: web-py-jinja2
title: 在web.py中使用jinja2模板
wordpress_id: 121
categories:
- web
tags:
- jinja2
- web.py
---

官方cookbook有介绍在web.py中使用jinja2模板的方法,
[http://webpy.org/cookbook/template_jinja.zh-cn](http://webpy.org/cookbook/template_jinja.zh-cn)但是有个缺憾,
只能渲染同级目录下的文件,下面是web.py的jinja2使用的实现.

```python
class render_jinja:
    """Rendering interface to Jinja2 Templates

    Example:

        render= render_jinja('templates')
        render.hello(name='jinja2')
    """
    def __init__(self, *a, **kwargs):
        extensions = kwargs.pop('extensions', [])
        globals = kwargs.pop('globals', {})

        from jinja2 import Environment,FileSystemLoader
        self._lookup = Environment(loader=FileSystemLoader(*a, **kwargs), extensions=extensions)
        self._lookup.globals.update(globals)

    def __getattr__(self, name):
        # Assuming all templates end with .html
        #这里造成了只能使用同级目录的文件,而且时能是html后缀的文件
        path = name + '.html'
        t = self._lookup.get_template(path)
        return t.render
```

web.py只是提供了jinja2模板使用的简单封装,至于实际使用中还是要自己封装.

下面演示了自己编写的最简单的jinja2的使用方法

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import web
from jinja2 import Environment, FileSystemLoader

urls = (
    '/index', 'index',
    '/admin', 'admin',
)

app = web.application(urls, globals())
#初始化jinja2的运行环境,使用FileSystemLoader
env = Environment(loader = FileSystemLoader('templates'))

class index(object):
    def GET(self):
        #取得同级目录下的文件
        return env.get_template('index.html').render()

class admin(object):
    def GET(self):
        #取得下层目录的文件
        return env.get_template('admin/admin.html').render()

if __name__ == '__main__':
    app.run()
```

更详细的使用方法可以去jinja2的官网查[http://jinja.pocoo.org/docs/](http://jinja.pocoo.org/docs/)
