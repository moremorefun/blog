---
author: admin
comments: true
date: 2011-10-01 02:32:48+00:00
layout: post
slug: webpy-form
title: web.py中使用form验证表单字段
wordpress_id: 139
categories:
- web
tags:
- form
- web.py
- python
---

在请求中验证用户输入是否合法是个比较麻烦的事情,
但其实`web.form`已经为我们提供了这方便的函数可以直接调用.

我们仅仅需要定义出一个form和每个字段的验证规则,
其他的事情`web.form`就自动替我们完成了.

下面是个十分简单的例子

```python
import web
from web import form
urls = (
    '/', 'hello',
)
app = web.application(urls, globals())
#定义字段验证规则
vpass = form.regexp(r".{3,20}$", 'must be between 3 and 20 characters')
#定义form表格内容
t_form = form.Form(
        form.Textbox('u',vpass,description='username'),
        form.Textbox('p',vpass,description='password')
)
class hello:
    def GET(self):
        """GET方法返回一个Form表格"""
        return """<html>
    <body>
        <form action="/" method="post">
            <p>user name: <input type="text" name="u" /></p>
            <p>password : <input type="password" name="p" /></p>
            <input type="submit" value="Submit" />
        </form>
    </body>
</html>
"""
    def POST(self):
        #生成一个form实例
        f = t_form()
        #返回form验证是否通过
        return f.validates()
if __name__ == "__main__":
    app.run()
```

更多`form`用法可以参考文档
