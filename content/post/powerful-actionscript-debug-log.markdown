---
author: admin
comments: true
date: 2013-08-02 03:14:35+00:00
layout: post
slug: powerful-actionscript-debug-log
title: 非常强悍的actionscript debug和log工具
wordpress_id: 114
categories:
- actionscript
tags:
- debug
- log
---

[flash-console](http://code.google.com/p/flash-console/)  这是一个用于调试swf的类库,用于添加到舞台上使用
使用方法非常的简单

```actionscript
import com.junkbyte.console.Cc;
Cc.startOnStage(this, "");
```

以上代码就能让调试面板在舞台上显示出来,
如果想在面板上输出log信息,可以用一下语句

```actionscript
Cc.config.commandLineAllowed = true // Enables full commandLine features
Cc.config.tracing = true; // also send traces to flash's normal trace()
Cc.config.maxLines = 2000; // change maximum log lines to 2000, default is 1000
Cc.config.commandLineAutoScope = true;  //自动切换命令行当前域(比较有用)
```

还可以根据自己的需要对调试面板进行一下属性设置

```actionscript
//start之前的配置
Cc.config.commandLineAllowed = true // Enables full commandLine features
Cc.config.tracing = true; // also send traces to flash's normal trace()
Cc.config.maxLines = 2000; // change maximum log lines to 2000, default is 1000

Cc.startOnStage(this); // finally start with these config

//start之后的设置
Cc.commandLine = true;
Cc.width = 460;
Cc.height = 500;
```

下面是命令行的使用

+ 默认的scope是stage,在命令行输入width,回车可以打印stage的width属性
+ 输入过程是有提示的,按tab可以实现自动补全
+ 另外使用`/command`命令可以查看内置的一下命令

我们还可以查看和调用自身内部对象的属性和函数,
例如我们有自己的类

```actionscript
class InnerData {
    public var va:int;
    public var vb:String;

    public function InnerData() {
        va = 10;
        vb = "hello word";
        }

    public function sum(a:int, b:int):int {
        return a + b;
    }

}
```

首先要注册带Cc中

```actionscript
Cc.store("inner", new InnerData ());
```

在运行过程中,命令行中输入`$inner`,回车

这时候会切换到我们的内置数据结构中,我们可以通过代码调用sum函数

```actionscript
sum(10,19)
```

命令行会返回给我们结果.

还有很多强悍的功能,例如watch,官网写的都很详细

[演示程序](http://console.junkbyte.com/sampleAdvanced.swf)
