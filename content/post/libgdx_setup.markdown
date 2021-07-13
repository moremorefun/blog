---
author: admin
comments: true
date: 2013-07-20 14:24:35+00:00
layout: post
slug: libgdx_setup
title: libgdx工程搭建(1)
wordpress_id: 160
categories:
- libgdx
tags:
- 环境搭建
---

首先,请确保安装好了android基于eclipse的开发环境:

- eclipse
- Android SDK
- ADT这些工具

libgdx的代码是存放在Google code的,
地址是:[libgdx](http://code.google.com/p/libgdx/),
如果耐心的看英文的话这里有大部分你需要的东西.

前不久有位大哥为我们写了一个libgdx的部署工具,
着实方便了大家新建libgdx项目.
使用方法原文在这里:[ProjectSetupNew](http://code.google.com/p/libgdx/wiki/ProjectSetupNew).

大概说明一下这篇wiki的内容

这个部署工具被打包成了可执行的jar文件:[gdx-setup-ui.jar](http://libgdx.badlogicgames.com/nightlies/dist/gdx-setup-ui.jar)

下载以后在命令行运行:

```shell
java  -jar  gdx-setup-ui.jar
```

远行争取的话会弹出ui的libgdx工程创建工具了,如果不成功,检查一下path之类的是不是配置正确.

其中libgdx开发包的下载地址是[libgdx download](http://code.google.com/p/libgdx/downloads/list).

我这里用的是0.9.4版本

在创建完成后,可以看见在创建的目录下有右侧显示的几个文件夹,
现在打开eclipse, `import->existing project inti workspace`,导入这几个文件夹.

我没有创建html工程,导入后又三个工程:

- 核心的工程 `("my-gdx-game")`,
包含 所有的程序代码,作为源代码被链接到其他的工程,今后的编程大部分在这里.

    由于libgdx想实现在pc和android上相同的测试效果,
所以desktop和android工程可以理解为一个壳.
而核心的功能代码在这里,外壳只是一个启动器.
启动以后交给这个核心工程处理.

- desktop工程 `("my-gdx-game-desktop")`

    桌面工程,在pc上直接运行的工程,测试工作大部分可以在这里完成

- 和android工程 `("my-gdx-game-android")`

这时候运行desktop和android工程,如果没问题,证明创建成功了.

之后介绍一个这几个工程的文件布局,以便可以手动生成自己需要的工程文件:
原文如下:[wiki/ProjectSetup](http://code.google.com/p/libgdx/wiki/ProjectSetup).

首先,核心工程是一个java工程,在libs中需要包含 gdx.jar文件

其次,desktop工程,也是一个java工程,需要包含

- `gdx-natives.jar`,
- `gdx-backend-lwjgl.jar`
- `gdx-backend-lwjgl-natives.jar`

另外就是要添加核心工程这个工程依赖了.

之后是android工程,这个要新建的就不能是java工程了,而是一个标准的android工程,
需要添加`gdx-backend-android.jar`依赖.

另外值得说明的是,libgdx是一个整合了android ndk的引擎.
我们虽然不用安装ndk开发环境,
但是需要把开发包中编译好的链接文件添加到android工程中的libs目录中,
也就是zip文件中的`armeabi`和`armeabi-v7a`目录.

这里简略的说明了一下wiki上搭建libgdx开发环境的文章,更详细的内容可以参考wiki.
