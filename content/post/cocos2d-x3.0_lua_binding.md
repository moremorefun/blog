---
layout: post
title: cocos2d-x 3.0 lua绑定项目资源拷贝过程
date: 2014-10-01 19:57:22
categories:
- cocos2d-x
tags:
- lua
- copy
- script
---

3.0rc0版本的cocos2d-x创建的lua版本与以往的版本文件夹结构不同:

- frameworks
    - cocos2d-x         cocos2d-x的源文件
    - runtime-src
        - Classes       项目c++源文件
        - proj.android  android平台工程文件
        - proj.ios_mac  ios_mac平台工程文件
        - proj.win32    win32平台工程文件
        - proj.linux    linux平台工程文件
- res                   资源文件
- runtime
- src                   lua源文件

并且编译过程中利用了编译脚本做了一些资源素材的拷贝工作,例如吧src文件夹下的lua脚本和res文件夹下的素材文件拷贝到工程资源目录下.

其中win32平台的编译脚本在vs的工程设置中可以看到:

![vs_copy_script](/images/vs_copy_script.png)

```shell
if not exist "$(OutDir)" mkdir "$(OutDir)"
if exist "$(OutDir)\Resource" rd /s /q "$(OutDir)\Resource"
mkdir "$(OutDir)\Resource"
mkdir "$(OutDir)\Resource\src"
mkdir "$(OutDir)\Resource\res"
xcopy "$(ProjectDir)..\..\cocos2d-x\cocos\scripting\lua-bindings\script" "$(OutDir)\Resource" /e /Y
xcopy "$(ProjectDir)..\..\..\src" "$(OutDir)\Resource\src" /e /Y
xcopy "$(ProjectDir)..\..\..\res" "$(OutDir)\Resource\res" /e /Y
```

把src拷贝到`输出目录/Resource/src`下,
把res拷贝到`输出目录/Resource/res`下,
把cdx框架中的lua源文件拷贝到`输出目录/Resource`下.

对应的android工程的拷贝配置文件为`frameworks/runtime-src/proj.android/build-cfg.json`.

```json
{
    "ndk_module_path" :[
        "../../cocos2d-x",
        "../../cocos2d-x/cocos/",
        "../../cocos2d-x/external",
        "../../cocos2d-x/cocos/scripting"
    ],
    "copy_to_assets" :[
        "../../../src",
        "../../../res",
        "../../cocos2d-x/cocos/scripting/lua-bindings/script/"
    ]
}
```

可以通过修改目录结尾是否带`/`符来控制复制的是文件夹还是文件夹下的所有文件.

在ios工程中由于资源路径可以和文件夹路径无关,所以工程中直接引入了,不需要拷贝.

了解了以上复制规则,我们就可以根据自己的需要更改拷贝目录的规则了.

比如我想向以前一样,不需要把lua的源文件和资源文件分为不同的文件夹,放在Resource同级就好了.那么我们可以修改win32的脚本为:

```shell
xcopy "$(ProjectDir)..\..\..\src" "$(OutDir)\Resource" /e /Y
xcopy "$(ProjectDir)..\..\..\res" "$(OutDir)\Resource" /e /Y
```

android的修改为

```json
{
    "ndk_module_path" :[
        "../../cocos2d-x",
        "../../cocos2d-x/cocos/",
        "../../cocos2d-x/external",
        "../../cocos2d-x/cocos/scripting"
    ],
    "copy_to_assets" :[
        "../../../src/",        //带分隔符表示文件夹下的文件,而不是文件夹
        "../../../res/",
        "../../cocos2d-x/cocos/scripting/lua-bindings/script/"
    ]
}
```

或者你可以根据自己的需要用luajit编译自己的lua文件,等等操作都可以放在这些脚本里.
