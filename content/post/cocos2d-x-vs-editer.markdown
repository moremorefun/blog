---
author: admin
comments: true
date: 2013-11-28 12:39:39+00:00
layout: post
slug: '%e4%bd%bf%e7%94%a8vs%e4%bd%9c%e4%b8%bacocos2d-x%e7%9a%84%e7%bc%96%e8%be%91%e5%99%a8%e7%9a%84%e5%87%a0%e7%82%b9%e6%b3%a8%e6%84%8f%e4%ba%8b%e9%a1%b9'
title: 使用vs作为cocos2d-x的编辑器的几点注意事项
wordpress_id: 294
categories:
- cocos2d-x
tags:
- vs
- ide
- 开发环境
---

1. 建议给vs安装Visual Assist X插件

    插件提供了文件查找,符号查找,代码模板,高级自动完成等功能

2. 使用`tools/project-creator/create_project.py`创建多平台工程

    使用vs打开`projects/(appname}\proj.win32下的{appname}.sln`解决方案文件.

3. 通过vs的"解决方案管理器"中邮件解决方案可以导入默认未添加,但是你需要的功能,可以在cdx提供的相应文件夹中找到对应的`*.vcxproj`文件

    并在你编辑的工程文件属性中添加为依赖项  

4. 如果遇到找不到头文件的问题  

    右键编辑的`工程文件->属性->c++->常规->附加目录`

    查找到需要的头文件的路径加进去.
