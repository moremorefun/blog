---
layout: post
title: cocos2d-x启用luajit的5.2特性
date: 2014-10-29 07:57:52+00:00
categories:
- cocos2d-x
tags:
- lua
- luajit
---

cocos2d-x中使用的luajit的库是`预编译`也就是`prebuilt`的,
但是这个预编版本并没有启用5.2的特性,也就是说

+   `pairs() and ipairs() check for __pairs and __ipairs`

这个特性是不被支持的.

我需要在项目中使用 [`cloudwu/pbc`](https://github.com/cloudwu/pbc)
这个protobuf的lua解析库,这在个库中,用到了5.2的这个特性

查看`pbc`的源码 [__pairs使用源码](https://github.com/cloudwu/pbc/blob/master/binding/lua/protobuf.lua#L523)

```lua
local function expand(tbl)
    local typename = rawget(tbl , 1)
    local buffer = rawget(tbl , 2)
    tbl[1] , tbl[2] = nil , nil
    assert(c._decode(P, decode_message_cb , tbl , typename, buffer), typename)
    setmetatable(tbl , default_table(typename))
end

function decode_message_mt.__index(tbl, key)
    expand(tbl)
    return tbl[key]
end

--调用pairs的时候会调用expand函数,用于解析下一层消息
function decode_message_mt.__pairs(tbl)
    expand(tbl)
    return pairs(tbl)
end
```

从源码中我们可以看出,pbc对于protobuf的解析并不是全部解析的,
也就是说,遇到message嵌套的情况,初始只
解析一层消息,
当代码调用下一层消息的`__index`和`__pairs`元函数的时候,
才会进行下一层消息的实际解析

那么问题来了,当这个库用在`cocos2d-x`上的时候,我们在

```lua
for k, v in pairs( msg ) do
```

的时候,是不会调用到expand方法的,这样获取的消息内容当然也就不正确了

为了能顺利的使用`pbc`库,我们需要启用luajit的这个特性

根据luajit官网说明[lujit/extensions](http://luajit.org/extensions.html),
可以通过编译时加入预定义`LUAJIT_ENABLE_LUA52COMPAT`宏来启用这一特性.

以`cocos2d-x 3.0rc0`版本为例,luajit的源码路径为`cocos2d-x-3.0rc0/external/lua/luajit`,
使用的luajit版本为`2.0.1-hotfix`,源码中已经打好了hotfix的包.

并且在根目录下提供了

+   `build_android.sh`
+   `build_ios.sh`
+   `build_mac.sh`

三个编译脚本用来生成不同平台的预编译库

但是,恩,`但是`,源码里不知道为何把`Makefile`文件删除了.

我们需要去官网下载`2.0.1`版本的`luajit`源码,然后把两个`Markfile`文件,
拷贝到cdx对应的目录下. 注意,有两个`Markfile`,`luajit`源码的根目录下一个,
`src`目录下一个.

##首先在windows平台下进行编译

windows平台的编译脚本是`cocos2d-x-3.0rc0/external/lua/luajit/src/src/msvcbuild.bat`

1.  修改`msvcbuild.bat`文件,找到`@set LJCOMPILE=cl /nologo /c /MD /O2 /W3 /D_CRT_SECURE_NO_DEPRECATE`,
添加编译参数,更改为`@set LJCOMPILE=cl /nologo /c /MD /O2 /W3 /D_CRT_SECURE_NO_DEPRECATE /DLUAJIT_ENABLE_LUA52COMPAT`
2.  不能在命令行直接调用这个bat脚本,因为需要vs编译环境参数,打开vs的命令行工具.
![命令行工具](/images/cdx_luajit_vs_tool.png)
![命令行工具](/images/cdx_luajit_vs_cmd.png)
3.  切换到编译脚本所在路径,运行`msvcbuild.bat`完成编译

编译成功后会生成

+   `luajit.exe`    用于编译lua脚本的工具
+   `lua51.dll`     库文件
+   `lua51.lib`     库文件

##windows版本的编译没有使用Makefile文件,但是其他版本的编译都使用了Makefile文件,所以我们直接把编译参数在Makefile里更改就可以了

在`cocos2d-x-3.0rc0/external/lua/luajit/src/src`中找到刚改从`luajit`源码中拷贝过来的`Makefile`.
找到`#XCFLAGS+= -DLUAJIT_ENABLE_LUA52COMPAT`,把注释取消,`XCFLAGS+= -DLUAJIT_ENABLE_LUA52COMPAT`.

##ios版本的编译

打开`build_ios.sh`文件,根据自己xcode的版本修改成自己的参数

```shell
#!/bin/sh
DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
LIPO="xcrun -sdk iphoneos lipo"
STRIP="xcrun -sdk iphoneos strip"

SRCDIR=$DIR/src
DESTDIR=$DIR/prebuilt/ios
IXCODE=`xcode-select -print-path`
ISDK=$IXCODE/Platforms/iPhoneOS.platform/Developer
ISDKVER=iPhoneOS7.1.sdk   #xcode中iPhone sdk的版本
ISDKP=/usr/bin/           #gcc所在路径,注意,低版本的可能在XCode路径下
```

然后执行就好了

##mac版本的编译和ios差不多,而且不用更改编译脚本

##android版本的编译

首先要在系统上安装ndk,并且设置`NDK_ROOT`环境参数,因为编译脚本要用到

打开`build_android.sh`文件,修改相应的参数

```shell
host_os=`uname -s | tr "[:upper:]" "[:lower:]"`      #要注意host_os的设置,因为如果这windows上使用cygwin编译,
```
这个参数需要更改为windows


```shell
NDK=$NDK_ROOT
NDKABI=8
NDKVER=$NDK/toolchains/arm-linux-androideabi-4.6
NDKP=$NDKVER/prebuilt/${host_os}-x86/bin/arm-linux-androideabi-     #需要注意-x86,如果使用的是64位的ndk,需要更改为-x86_64
NDKF="--sysroot $NDK/platforms/android-$NDKABI/arch-arm"
```

```shell
# Android/x86, x86 (i686 SSE3), Android 4.0+ (ICS)
NDKABI=14
DESTDIR=$DIR/prebuilt/android/x86
NDKVER=$NDK/toolchains/x86-4.6
NDKP=$NDKVER/prebuilt/${host_os}-x86/bin/i686-linux-android-        #同上,根据ndk的版本进行修改
```

编译后就能获得android版本的库了

所有库编译完成后,我们就可以在各个平台的luajit中支持5.2的这些特性了
