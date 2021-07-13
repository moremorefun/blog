---
layout: post
title: 在cocos2d-x 3.0 中使用Dragonbones
date: 2014-10-10 23:37:44
categories:
- cocos2d-x
tags:
- 3.0
- 中使用Dragonbones
---

`DragonBones`官方已经提供了cocos2d-x的库文件[SkeletonAnimationLibraryCPP](https://github.com/DragonBones/SkeletonAnimationLibraryCPP),
目前支持到了2.2.
但是并未提供3.0版本的库,官方说3.0发布正式版时会支持.

如果现在就要使用3.0版本的话稍作修改也是可以实现的.

另外需要注意的是,在写文章时,这个库的master分支貌似有bug,需要使用dev分支的源码修改.

以下`DragonBones`用`db`表示.

在项目的wiki中已经说明,平台有关的文件只有以下几个:

- Cocos2dxAtlasNode.cpp         `db`中使用使用的基本现实对象,类似`cocos2d-x`中的`Node`
- Cocos2dxDisplayBridge.cpp     `db`和`cocos2d-x`现实对象的操作中间件
- Cocos2dxFactory.cpp           `db`的生成工厂
- Cocos2dxTextureAtlas.cpp      `db`读取texture的类

其实对于`Cocos2dxAtlasNode.cpp`可以用`Sprite`代替,
`Cocos2dxTextureAtlas.cpp`只用用`SpriteFrameCache`就好了.

所以这两个类不用改,删掉就好了.

为了实现这一目的,我们需要在`db`导出素材时选择`Zip(XML+PNGS)`,
这是为了我们可以使用`TexturePacker`来打包图片素材,用`SpriteFrameCache`加载.
`TexturePacker`的打包还是很让人称赞的.

![db导出选项](/images/dragenbones_export_setting.png)

导出以后解压zip,把`texture`文件夹下的图片用`TexturePacker`打包到成plist就可以用一下函数加载了:

```c++
SpriteFrameCache::getInstance()->addSpriteFramesWithFile("xxx.plist");
```

下面修改`Cocos2dxFactory.cpp`.

- `CC`前缀的修改
- `CCNode.h`等头文件路径的修改

主要是这个函数:

```c++
//这个函数用来根据db的数据生成cocos2d-x的显示对象
Object* Cocos2dxFactory::generateDisplay(ITextureAtlas *textureAtlas, const String &fullName, Number pivotX, Number pivotY)
{
    std::stringstream ss;
    ss << fullName << ".png";

    //用资源名直接用缓存里生成Sprite
    auto sp = cocos2d::Sprite::createWithSpriteFrameName(ss.str());
    auto size = sp->getContentSize();
    sp->setCascadeOpacityEnabled(true);
    //设置sp的注册点,注意Y方向,flash和cocos2d-x的坐标系不同
    sp->setAnchorPoint(cocos2d::Point(
        pivotX / (float)size.width,
        1.0f - pivotY / size.height
        ));
    return new CocosNode(sp);
}
```

`Cocos2dxDisplayBridge.cpp`基本只用修改头文件和`CC`前缀就好了,
这个文件只要勇于操作`cocos2d-x`的显示对象的各种变换和添加移除.

删除其余的`Cocos2dx`开头的文件,修复各种编译错误.

然后就可以用了

```c++
SpriteFrameCache::getInstance()->addSpriteFramesWithFile("Zombie/t.plist");

dragonBones::Cocos2dxFactory fac;
dragonBones::XMLDataParser parser;
dragonBones::XMLDocument doc;

doc.LoadFile("Zombie/skeleton.xml");
fac.addSkeletonData(parser.parseSkeletonData(doc.RootElement()));

_zombie = fac.buildArmature("Zombie_polevaulter", "", "Zombie");
auto skeletonNode = static_cast<dragonBones::CocosNode*>(_zombie->getDisplay())->node;
skeletonNode->setPosition(50, 300);
addChild(skeletonNode);
_zombie->getAnimation()->gotoAndPlay("anim_walk");
```

因为`db`结构很好,和平台相关的类封装的都很好,
所以修改起来很方便.

其实我们只不过修改了`generateDisplay`函数,改为使用`Sprite`这样避免了对`Cocos2dxAtlasNode`的使用.
利用了`cocos2d-x`自己的素材加载函数,避免了对`Cocos2dxTextureAtlas`的使用.

额外一些工作就是删除文件,修改头文件路径,修改编译错误.
