---
author: admin
comments: true
date: 2012-09-25 13:02:50
layout: post
slug: asunit-asynchronous-test
title: asunit异步测试
wordpress_id: 247
categories:
- actionscript
tags:
- asunit
- Async
---

asunit是actionscript的一个单元测试框架,
对于同步执行的函数不用多说,
网上能查到很多资料.

但是actionscript里存在大量的异步函数,
如`load`,`dispatchEventListener`.

如果涉及到这些异步的函数应该怎样测试呢?

例如

```actionscript
public class Tmp
{
    public var _bitmap:Bitmap;
            private var _func:Function;
    public function Tmp()
    {

    }

    public function loadRes(func:Function):void {
        var loader:Loader = new Loader();
        loader.loaderInfo.addEventListener(Event.COMPLETE, loaded);
                    _func = func;
        loader.load(new URLRequest("test.jpg"));
    }

    private function loaded(e:Event):void
    {
        _bitmap = (e.target as LoaderInfo).loader.content;
    }
    }
```

我们怎么测试loadRes有没有正确执行呢?

其实asunit已经为我们提供了异步测试函数`addAsync`

我们只需要创建一个TestCase

```actionscript
public class AssetManagerTest extends TestCase
{
    private var _tmp:Tmp;

    public function AssetManagerTest(methodName:String=null)
    {
        super(methodName);
    }

    override protected function setUp():void {
        super.setUp();
        _tmp= new Tmp();
    }

    override protected function tearDown():void {
        super.tearDown();
    }

    public function testLoadRes():void {
        _tmp.loadRes(addAsync(checkLoadRes));
    }

    ////////////////////////////////////////////
    private function checkLoadRes():void {
        assertNotNull(_tmp._bitmap);
    }
}
```

需要注意的是,我们的回调函数一定要用asunit的addAsync包装一下.
这样asunint才会等待一定的时间执行检测函数,否则起不到测试效果.

addAsync具体用法可以看一下源码,还有一些参数可用
