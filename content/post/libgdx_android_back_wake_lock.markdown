---
author: admin
comments: true
date: 2013-07-28 16:01:13+00:00
layout: post
slug: libgdx_android_back_wake_lock
title: libgdx于android的返回和屏幕唤醒(5)
wordpress_id: 244
categories:
- libgdx
tags:
- 退出程序
---

在libgdx中默认对返回按键的处理是退出程序,
如果我们希望加一个退出确认框,
那如何截获返回按键呢?

首先要知道`InputProcessor`接口,
这个接口定义了很多输入处理函数,如:

- 按键按下
- 点击屏幕
- 拖动
- 等等

InputProcessor使用方式如下:

```java
Gdx.input.setInputProcessor(inputProcessor);
```

所以我们需要自己实现一个InputProcessor,
然后设置一下就行了.

那么还有一个问题,
Stage实际上已经实现了InputProcessor,
因为他要处理Actor的点击之类的事情.
那么我们既想保持Stage作出输入处理类,
又想实现自己对输入的一些控制怎么办呢?

- 继承Stage,复写需要变更的方法
- 利用多重输出处理类InputMultiplexer

```java
InputMultiplexer multiplexer = new InputMultiplexer();
multiplexer.addProcessor(new MyUiInputProcessor());
multiplexer.addProcessor(new MyGameInputProcessor());
Gdx.input.setInputProcessor(multiplexer);
```

作为我们来讲,一个processor设置成stage,一个设置成自己的实现就可以了.

另外我们需要手动设置input截获返回按键

```java
Gdx.input.setCatchBackKey(true);
```

截获菜单按键也是一样的,如果不手动设置,系统会自己处理掉.

现在复写InputProcessor的keyUp方法,因为keyDown如果按住不放的话会一直调用

```java
@Override
public boolean keyUp(int keycode) {
    //判断按下的是返回按键
    if(Input.Keys.BACK == keycode){
        //打印一句log
        Gdx.app.log("s", "back key typed");
        //这里就是推出应用,当然可以定义自己的处理
        Gdx.app.exit();
    }
    return false;
}
```

下一个,怎么在游戏中保持屏幕唤醒?

在Android项目的主文件中可以看到初始化的时候用到了`AndroidApplicationConfiguration`,
有一个属性就是是否保持屏幕唤醒useWakelock,设置为true.还有其他一下选项,可以看看api.

但是还有一个重要的东西,
Android的很多功能都是需要在配置文件中申请权限的,
这个屏幕唤醒也是其中一项:需要在AndroidManifest.xml配置:

```xml
<uses-permissionandroid:name="android.permission.WAKE_LOCK" />
```

这个选项和application同级.

加上这个权限配置才能真正实现屏幕保持唤醒.
