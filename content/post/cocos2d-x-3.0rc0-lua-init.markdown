---
layout: post
title: cocos2d-x3.0rc0版本lua的初始化过程中的一些问题
date: 2014-10-17 07:57:52+00:00
categories:
- cocos2d-x
tags:
- lua
- 初始化
---

首先调用的是`LuaEngine`的init()方法,
在`LuaEngine::init`首先初始化了`LuaStack`,
然后去加载被被标记为过时的lua api库

其实主要的lua初始化过程在`LuaStack`中

`LuaStack::init`中首先创建luastate,
然后加载基础类库,
之后加载`tolua++`的fix.

再之后就开始注册cocos2d-x在lua中所能使用的函数了

首先注册的是`print`函数,这里需要特别注意
*`luaL_register`的调用会在lua栈中push进`_G`这个table*

在之后register_all_xxx的时候会调用一个`tolua_module`的函数,
而这个函数会用到栈顶的`_G`这个table.

```c++
if (name)
{
    /* tolua module */
    lua_pushstring(L,name);
    lua_rawget(L,-2); //看见这里了吗?这里实际上是在luaL_register时候添加到栈里的
    if (!lua_istable(L,-1))  /* check if module already exists */
    {
        lua_pop(L,1);
        lua_newtable(L);
        lua_pushstring(L,name);
        lua_pushvalue(L,-2);
        lua_rawset(L,-4);       /* assing module into module */
    }
}
```

*这也就是为什么我们在自己的函数中使用相同的方式`register_all_xxx`的时候会报错的原因*,
因为`_G`已经不在栈里了

其他部分都可以通过读源码了解,但是这个`tolua_module`对`_G`表的使用方式感觉多少有些不恰当
