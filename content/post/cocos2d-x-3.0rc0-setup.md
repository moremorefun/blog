---
layout: post
title: cocos2d-x3.0rc0的setup.py脚本
date: 2014-10-17 07:57:53+00:00
categories:
- cocos2d-x
tags:
- setup.py
- 3.0rc0
---

这个脚本的主要作用就是设置cocos2d-x所需要用到的环境变量,
其中包括:

+   `COCOS_CONSOLE_ROOT`   cocos2d-x的各种脚本的路径
+   `NDK_ROOT`
+   `ANDROID_SDK_ROOT`
+   `ANT_ROOT`     ant工具的路径,bin文件夹的路径

`COCOS_CONSOLE_ROOT`是不需要手动添加的,脚本会自动检测其路径
如果你的环境中已经存在了如上路径,可以忽略相应的参数

+   如果当前环境为win32,则修改注册表,添加环境变量
+   如果不是,则把变量设置数据写入`.bash_profile``bash_login``.profile`这些相应的文件

至于生效的问题

+   所以无论在win32环境需要重启命令行使环境变量生效
+   在unix环境需要source相应的文件使环境变量生效,下一次启动终端将自动执行这些命令