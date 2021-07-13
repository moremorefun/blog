---
author: admin
comments: true
date: 2014-10-16 07:57:52+00:00
layout: post
slug: cocos2d-x%e7%9a%84textfield%e5%9c%a8windows%e4%b8%ad%e9%80%80%e6%a0%bc%e9%94%ae%e5%a4%b1%e6%95%88
title: cocos2d-x的TextField在windows中退格键失效
wordpress_id: 297
categories:
- cocos2d-x
tags:
- 退格失效
- TextField
---

在windows下使用TextField控件,退格键是不起作用的
这是因为CCGLView.cpp中通过监听GLFWChar来操作字符
但是退格键左右操作字符是不在这个函数回调的
但是CCGLView监听的GLFWKey并没有调用IMEDDispatch的dispatchDeleteBackward函数用于处理退格
是为了让我们自己控制退格键的作用?
解决方案有两个
1.修改CCGLView源码,在GLView::onGLFWKeyCallback函数中添加

```c++
if (GLFW_KEY_BACKSPACE == key)
{
    if (GLFW_REPEAT == action || GLFW_PRESS == action)
    {
        IMEDispatcher::sharedDispatcher()->dispatchDeleteBackward();
    }
}
```

2.自己监听KeybordEvent,在退格处理用调用IMEDispatcher::sharedDispatcher()->dispatchDeleteBackward()
但是第二种方法的问题是不能处理按键的GLFW_REPEAT事件,也就是一直按下退格无法连续删除文字
