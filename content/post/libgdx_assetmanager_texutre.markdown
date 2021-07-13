---
author: admin
comments: true
date: 2013-07-25 16:19:30+00:00
layout: post
slug: libgdx_assetmanager_texutre
title: libgdx素材加载(3)
wordpress_id: 217
categories:
- libgdx
tags:
- assetmanager
- texutre
---

关于素材,我们需要知道的一个基本的类是`Texture`,
它是一张图片在程序内存中的映像.

如果需要程序显示一张图片,那么就需要把它转化为一个Texture的实例,
有一个叫`SpriteBatch`的东西知道怎么把它绘制在屏幕上.

而这个`Texture`对加载的图片是有要求的长宽都必须是2的整数次幂.
很显然,我们实际中使用的素材不可能原本就是这个大小.
于是我们会把几张实际使用的图片,拼到一张1024*1024的图片上.
当然,有工具替我们完成这些,所以不用着急,先理解一下这些概念.

这样一来,我们可能在程序中使用的不是整个`Textrue`,而是`Textrue`中的某一个区域.
于是便有了`TextureRegion`,可以把它理解为Texture的一个装饰类,
内部持有一个`Texture`实例,记录了`Texture`的裁剪信息.
以便我们标记需要使用的区域,
而`SpriteBatch`也提供了绘制`TextureRegion`的效用,之绘制这个裁剪出来的区域.

那么既然有工具来把很多图片拼凑到一张图片上的工具,
[libgdx-texturepacker-gui](http://code.google.com/p/libgdx-texturepacker-gui/).

理论上我们应该也不用自己去计算原始图片在这张大图上的位置信息,
实际上这个 工具生成了两个文件,一个裁剪信息文件,一个合并好的png文件.

这样一来,我们应该也不用自己写代码解析这个配置文件吧.

是的,libgdx提供了TextureAtlas这个类用来解析这个配置文件
加载大的png文件,并切割成对应的TextureRegion供我们使用.

当然,你可以把图片都弄成2的整次幂,
但是绝对不提倡,因为我猜测,libgdx的内存和cocos2d是相同的,
因为Texture占用的内存和它存储的图片大小有关.

例如3*3的Texture实际上占用了8*8的空间.

我这里已经下载好了`texturepacker-gui`这个工具,
这个工具会:

- 把输入目录中的图片文件
- 拼接到一张或多张png文件中
- 生成一个配置文件

你可以大致阅读一下这个配置文件,
一般叫`pack`,
里面记录了生成的png文件们的路径,
拼合的文件的标识符,也就是原始图片的文件名,
以及他们的剪切信息.

在解压以后的目录里有一个`test-me`,
我这里直接使用它的输出文件测试.

把`test-me`里out文件夹下的文件复制到android工程的assets文件夹下,
于是那里多了两个文件input1.png和pack.

刷新这个工程,否则程序可能无法读取到这两个文件,很神奇.

如果pack文件和png文件在同一目录下,我们创建atlas的时候只需传递pack的路径就可以了.
因为开源嘛,有兴趣可以看一下`TextureAtlas`的源码.

```java
//创建TextureAtlas,并读取assets文件夹下的pack文件
TextureAtlas ta = new TextureAtlas("pack");
//获取切割好的素材,"test01"是个标识名,可以在pack文件中看到,实际上就是打包之前的文件名
TextureRegion tr = ta.findRegion("test01");
//用切割好的素材创建一个Image对象
img = new Image(tr);
//添加到舞台上显示出来
stage.addActor(img);
```

以上简单的几行代码说明了TextureAtlas的使用方法.

现在了解了怎样加载一张图片,你可能会好奇,怎样获得加载进度呢?

libgdx的`AssetManager`不但为我们提供了异步加载和素材管理功能,
也为我们提供了获取加载进度的方法.
不过它提供的加载进度比较坑爹,
之后我们会介绍.
