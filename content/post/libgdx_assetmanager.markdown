---
author: admin
comments: true
date: 2013-07-28 14:50:34+00:00
layout: post
slug: libgdx_assetmanager
title: libgdx中的AssetManager(4)
wordpress_id: 230
categories:
- libgdx
tags:
- AssetManager
---

[http://code.google.com/p/libgdx/wiki/AssetManager](http://code.google.com/p/libgdx/wiki/AssetManager),
wiki里作者给我们列举了使用AssetManager的几种好处:

- 加载大部分资源使用异步的方式,这样就能在加载的同时不阻塞渲染进程
- 实现了引用计数,当A和B都依赖C素材的时候,C只有在A,B都销毁了才会销毁.这也意味着即使一个资源加载了很多次,在内存中也之后一份
- 使用一个单一的实力管理所有素材
- 可以实现素材的缓存

创建AssetManager的方法是

```java
AssetManager manager = new AssetManager();
```

当加载素材的时候,
AssetManager需要知道用哪个loader来加载这个素材.
这些loader都实现了AssetLoaders里的方法,
其中分为两类:

- `SynchronourAssetLoader`
- `AsynchronousAssetLoader`

前一种在渲染进程加载资源,
后面一种在另外的进程加载资源.

例如创建`Pixmap`需要Texture,
并且有些OpenGL素材依赖于渲染进程,
下面是系统已经实现的一些资源和loader的对应关系:

- Pixmaps via PixmapLoader
- Textures via TextureLoader
- BitmapFonts via BitmapFontLoader
- TextureAtlases via TextureAtlasLoader
- TiledAtlases via TiledAtlasLoader
- TileMapRenderers via TileMapRendererLoader
- Music instances via MusicLoader
- Sound instances via SoundLoader

使用方法是:

```java
manager.load("data/mytexture.png", Texture.class);
manager.load("data/myfont.fnt", BitmapFont.class);
manager.load("data/mymusic.ogg", Music.class);
```

这些资源将添加入加载队列,
如果需要在资源加载时使用特殊的初始化参数,
可以参照下面的方法:

```java
//加载TextureParameter所使用的参数
//每种资源所传递的加载参数类型是不一样的
TextureParameter param = new TextureParameter();
param.minFilter = TextureFilter.Linear;
param.genMipMaps = true;
//load时传入就可以了
manager.load("data/mytexture.png", Texture.class, param);
```

目前为止,素材只是添加到了加载丢列,而不会开始加载,我们需要逐帧调用
[AssetManager#update](http://code.google.com/p/libgdx/wiki/AssetManager#update)()函数

```java
public MyAppListener implements ApplicationListener {

   public void render() {
      if(manager.update()) {
         // updata()返回true,证明所有资源加载完成
         // 可以执行对应的操作了
      }

      // 获取加载进度,这个还是比较坑爹的,之后会说明
      float progress = manager.getProgress()
      ... left to the reader ...
   }
}
```

如果需要强制完成所有加载,可以调用如下函数.
意思是将异步加载专为同步加载,
这个函数会阻塞到队列中的素材加载完成.

```java
manager.finishLoading();
```

获取素材的方式是:

```java
Texture tex = manager.get("data/mytexture.png", Texture.class);
BitmapFont font = manager.get("data/myfont.fnt", BitmapFont.class);
```

判断某单一素材是否加载完成的方式是:

```java
if(manager.isLoaded("data/mytexture.png")) {
   // texture is available, let's fetch it and do do something interesting
   Texture tex = manager.get("data/mytexture.png", Texture.class);
}
```

销毁素材:

```java
manager.unload("data/myfont.fnt");
```

销毁所有素材的方式是:

```java
manager.clear();

//或者
//区别是dispose会把AssetManager也销毁掉
manager.dispose();
```

另外,我们可以自己定义文件的读取方式.
例如如果素材自己加密了,我们需要解密才能读取
这时候可以实现自己的FileHandleResolver.

里面就一个函数

```java
//根据文件名,返回一个FileHandle对象
public FileHandle resolve (String fileName);
```

new的时候传递进去就行了

```java
AssetManager manager = new AssetManager(new ExternalFileHandleResolver());
```

另外一个需要说明的是,当程序切换到后台,很有可能为了节省内存,
素材会被销毁.
那么程序切换到前台是需要重新加载这些资源?
使用AssetManager加载的资源无需担心,
但是如果不是,可以使用:

```java
Texture.setAssetManager(manager);
```

的方式来把这些资源让AssetManager管理.

另外,可能系统提供的loader不满足我们的需要,
那我们可以自己定义Loader来加载我们自己的资源.

从AssetManager的源码中我们可以看到他是怎么添加Loader的:

```java
public AssetManager (FileHandleResolver resolver) {
	setLoader(BitmapFont.class, new BitmapFontLoader(resolver));
	setLoader(Music.class, new MusicLoader(resolver));
	setLoader(Pixmap.class, new PixmapLoader(resolver));
	setLoader(Sound.class, new SoundLoader(resolver));
	setLoader(TextureAtlas.class, new TextureAtlasLoader(resolver));
	setLoader(Texture.class, new TextureLoader(resolver));
	setLoader(Skin.class, new SkinLoader(resolver));
	setLoader(TileAtlas.class, new TileAtlasLoader(resolver));
	setLoader(TileMapRenderer.class, new TileMapRendererLoader(resolver));
	//..........不用看的代码
}
```

如果需要定义自己的Loader可以参照这些Loader实现,
`MusicLoader`同步加载器

```java
public class MusicLoader extends SynchronousAssetLoader<Music, MusicLoader.MusicParameter> {
	public MusicLoader (FileHandleResolver resolver) {
		super(resolver);
	}

    //调用load时,返回需要创建的对象
	@Override
	public Music load (AssetManager assetManager, String fileName, MusicParameter parameter) {
		return Gdx.audio.newMusic(resolve(fileName));
	}

    //这里返回本素材的依赖数组
    //例如bitmapfont在配置文件外需要文字图片
    //那么可以在这里返回
	@Override
	public Array<AssetDescriptor> getDependencies (String fileName, MusicParameter parameter) {
		return null;
	}

	static public class MusicParameter extends AssetLoaderParameters<Music> {
	}
}
```

异步加载器`PixmapLoader`

```java
public class PixmapLoader extends AsynchronousAssetLoader<Pixmap, PixmapLoader.PixmapParameter> {
	public PixmapLoader (FileHandleResolver resolver) {
		super(resolver);
	}

	Pixmap pixmap;

        //非渲染进程调用函数
        //没有返回值
	@Override
	public void loadAsync (AssetManager manager, String fileName, PixmapParameter parameter) {
		pixmap = null;
		pixmap = new Pixmap(resolve(fileName));
	}

        //渲染进程调用函数
        //返回需要创建的对象,返回空证明还没加载完
	@Override
	public Pixmap loadSync (AssetManager manager, String fileName, PixmapParameter parameter) {
		return pixmap;
	}
        //同样,返回依赖项
	@Override
	public Array<AssetDescriptor> getDependencies (String fileName, PixmapParameter parameter) {
		return null;
	}

	static public class PixmapParameter extends AssetLoaderParameters<Pixmap> {
	}
}
```

ok,现在吐槽AssetManager的加载进度.

我如果调用AssetManager#load函数2次,
那么,返回的进度只会出现0,0.5,1.

也就是说AssetManager的进度完全不考虑依赖资源,
load几个,就计算几个.
因此,除非资源特别多,不要指望他返回的进度是平滑的.

在我看来这是很大的问题.....
