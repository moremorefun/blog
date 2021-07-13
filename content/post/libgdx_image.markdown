---
author: admin
comments: true
date: 2013-07-25 11:01:55+00:00
layout: post
slug: libgdx_image
title: 用libgdx显示一张图片(2)
wordpress_id: 189
categories:
- libgdx
tags:
- 显示图片
---

之前我们已经:

- 搭建好了libgdx开发环境
- 知道如何使用创建工具创建的工程
- 成功运行程序显示了一张图片

但是这种方式其实比较底层,可以看看一下核心工程中的ApplicationListener的实现类.

这里我会用一些更像flash和cocos2d的方式来显示一张图片.

打开核心工程的主文件,删除不需要的代码,剩下的内容据基本如下了:

```java
public class MyGdxGame implements ApplicationListener {

    //初始化会调用的函数
    @Override
    public void create() {
    }

    //销毁时调用的函数
    @Override
    public void dispose() {

    }

    //逐帧调用的函数
    @Override
    public void render() {
        Gdx.gl.glClearColor(1, 1, 1, 1);
        Gdx.gl.glClear(GL10.GL_COLOR_BUFFER_BIT);
    }

    //屏幕重置大小调用的函数
    @Override
    public void resize(int width, int height) {
    }

    //暂停时调用的函数
    @Override
    public void pause() {
    }

    //恢复到前台调用的函数
    @Override
    public void resume() {
    }
}
```

我们现在需要关注的两个函数是`create()`和`render()`

- 在create的时候我们创建需要显示的对象,然后把它添加到舞台
- 在render的时候我们清空屏幕,然后同时stage去重绘

现在我们介绍libgdx中几个类:

- Actor:演员,可以理解为一个显示对象,可以被添加到舞台上,但是需要自己实现draw方法
- Group:Actor的子类,实现了draw方法,可以被当作Actor的容器,会自动绘制容器中的Actor们
- Stage:Group的子类,可以理解为一个顶级的显示对象
- Image:Actor子类,libgdx为我们实现的显示图片的类

如果需要显示Actor,我们必须创建一个Stage用于绘制这些Actor,否则这些Actor无法显示出来
于是我们需要新建类变量,在create中创建Stage,在render中控制stage刷新和重绘.

```java
private Stage stage;
//初始化会调用的函数
@Override
public void create() {
        stage = new Stage(Gdx.graphics.getWidth(), Gdx.graphics.getHeight(), false);
}
//逐帧调用的函数
@Override
public void render() {
    Gdx.gl.glClearColor(1, 1, 1, 1);
    Gdx.gl.glClear(GL10.GL_COLOR_BUFFER_BIT);

    stage.act(Gdx.graphics.getDeltaTime());
        stage.draw();
}
```


至于Stage构造函数的参数到底是什么意思,可以看一下api:
[http://libgdx.badlogicgames.com/nightlies/docs/api/](http://libgdx.badlogicgames.com/nightlies/docs/api/).
这是每天更新的版本,在下载的zip文件中也有api.
那个是和你所使用的版本对应的,Stage的构造函数的参数这里就不具体介绍了.
我会用Image这个类来说明怎么使用libgdx的api文档.

好,现在我们有了一个舞台用于显示我们的图片,
那我们怎么把一张图片添加到舞台上呢?

我们打开api网页,
在"Package"一栏中找到[com.badlogic.gdx.scenes.scene2d.ui](http://libgdx.badlogicgames.com/nightlies/docs/api/com/badlogic/gdx/scenes/scene2d/ui/package-frame.html) 这个包,点击.

我们能在"Classes"栏中看到这个包里有哪些类.

根据经验,图片的类叫做"[Image](http://libgdx.badlogicgames.com/nightlies/docs/api/com/badlogic/gdx/scenes/scene2d/ui/Image.html)",点进去在右侧看看这个类的构造函数和为我们提供的方法.

Image的的构造函数大体够味两类,
一个以NinePath作为参数的,
一个以Texture或者TextureRegin作为参数的.
点击链接分辨看看这两个是做什么的.

在NinePath中没什么简介,
但是看到NinePath中有几个构造函数也是用Texture作为参数的.
回去看Texture的说明.

Texture:说明中说,这是一个标准的OpenGL ES的texture......
好吧,对于没用过OpenGL的人来说和没说一样.

google texture,意思是材质贴图,然后深入一下能了解到:
这其实相当于一个图片文件在程序内从中的映像,
也可以理解为图片在程序内存中的储存形式.

ok,那怎么得到这样一个Texture对象呢 ,继续看Textrue的构造函数.

有一个是以FileHandle作为参数的, 貌似是通过一个文件创建,
额,libgdx应该提供了文件读取功能吧?

去看看wiki,确实有, [http://code.google.com/p/libgdx/wiki/FileHandling](http://code.google.com/p/libgdx/wiki/FileHandling).
`Gdx.file.xxxx("路径")`,就返回一个FileHandle对象.

所以事情变得简单了,创建一个Image对象的方式就是:

```java
    //获得一个文件对象
    FileHandle imgfile = Gdx.files.internal("data/libgdx.png");
    //创建一个Texture对象
    Texture t = new Texture(imgfile);
    //创建一个图片Actor
    img = new Image(t, Scaling.none);
    //把它添加到舞台上
    stage.addActor(img);
```

把如上代码加入creat函数中,我们就成功的让一张图片显示在舞台上了.

需要注意的是,internal对应的是android工程中asset目录,
在文件处理的wiki中有说明,可以阅读一下.

我这里加载的这种图片是自动生成的工程里自带的,
libgdx对于加载的图片有要求,长宽必须是2的幂.
这张图片正好符合这个要求,如果换自己的图片,必须注意长宽,
否则会报错.

现在可能认为这个长宽的限制很大,
但是之后说到资源加载的时候我们会介绍一个工具,
用于把很多图片打包到一张大图里,
切割使用 所以现在不用担心.

那么总结一下:

- 我们这里使用的都是libgdx的高层类
- 底层的Sprite和SpriteBatch之类的暂时不管
- 如果你使用过flash或者接触过cocos2d可能更容易理解

我们使用一个舞台(Stage)作为显示对象的根节点.
然后在Stage上我们可以添加和移除各种Group,
Actor作为我们的显示对像并对他们进行控制来实现游戏逻辑展示.

这里重点告诉大家,学习使用libgdx的api文档,
之后会和大家详细介绍,资源加载
