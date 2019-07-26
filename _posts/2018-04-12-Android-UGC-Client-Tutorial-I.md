---
title: Android UGC Client Tutorial I
date: 2018-04-12 18:51:08
layout: post
description: "如何搭建 UGC 客户端"
category: "android"
tags: [ugc, android, Tutorial, arch, surface]
---

做一个 `Android` 平台的 `UGC` 客户端，需要涉及的领域很多，而每一个领域都有大量的知识需要积累。可这别无他法，唯有各个击破。

开篇文章主要从宏观上讲讲 Android 的图形架构，这样我们才能知道各个视频、画面等等是如何显示到屏幕上面去的。有的放矢，步子才能走得更坚定些。

<!--more-->

> 注1: 这是概述文章，只是为了大家能够对底层有一个大体的认识，这样在开发 UGC 客户端的时候，心里更有底。如果要完整叙述，怕得是一部分的工作量了。逃 :)

> 注2: 本人才疏学浅，难免很多错误，加之 Android 系统不断迭代改进，更不能保证文章的内容能一直适用。如有问题，欢迎指正，联系woaitqs@gmail.com。[博客](www.woaitqs.cc) 的评论功能，因为多说的关闭，还没有开放，读者朋友要是有合适的评论插件推荐，也希望不吝赐教。



#### 概述

![图形架构](http://o8p68x17d.bkt.clouddn.com/1.png)

Android的图形架构就是上面这张图，可能大家看过很多遍了，我在这里从小白的视角上再阐释一下。解决的问题是，如何把多种`Image Stream`，经过怎样处理后，显示到屏幕上去。

1. 生产者消费者模式
在系统的运行周期中，随时随地都在处理各种各样的数据，设想一下，如果同步地处理各个数据块，那势必相当卡顿，几乎没有容错率。解决方法是耳熟能详的生产者消费者模式，生产端和消费端解耦。在生产者和消费者，承担缓冲角色的就是 [BufferQueue](https://source.android.google.cn/devices/graphics/arch-bq-gralloc)。理解这个概念后，我们可以在大体上认为，是生产者生成某些 IMAGE STREAM，交由 BufferQueue，而在另一端消费者获取这些内容，并最终绘制到屏幕上去。(实际上比这个复杂得多，比如缓冲区的设置，且有多个 BufferQueue 等等)

2. IMAGE STREAM PRODUCERS
这些 `Image Stream`可能有多种来源，比如 MediaPlayer(视频)、OpenGL ES，Camera Preview, Canvas 等等。OpenGL ES 是一个针对移动平台上的图形渲染 API 标准，所以当然也能产生 `Image Stream`。通常情况下，应用程序的界面，都是通过 Canvas 进行操作。Canvas API 提供了一组软件实现方法(通常支持硬件加速，lockCanvas获取的Canvas目前不支持)，可以直接在 Surface 上绘图，是对 OpenGL ES 的低效替代品，因而 Canvas 也可以算作 `Image Stream` 中的一种。

3. Hardware Composer Layer
最终的消费端是硬件层，直接对接驱动对开发不太友好，Android 抽象出 Hardware Composer，方便系统开发者使用。SurfaceFlinger 可以将前面多个来源的 BufferQueue 进行合并处理，处理完毕后交由 `Hardware Composer` 显示到屏幕上去。深入了解这次，戳[SurfaceFlinger 和 Hardware Composer](https://source.android.google.cn/devices/graphics/arch-sf-hwc)。看看下图，可以方便理解。

![Android 图形管道](http://o8p68x17d.bkt.clouddn.com/2.png)

4. Surface
> Handle onto a raw buffer that is being managed by the screen compositor.
> A Surface is generally created by or from a consumer of image buffers (SurfaceTexture、MediaRecorder、Allocation)，and is handed to some kind of producer (eglCreateWindowSurface、MediaPlayer#setSurface、CameraDevice) to draw into.

说到这里，还有一个问题没有解答，就是 -- 怎么把数据传输给 BufferQueue？答案是 Surface，Surface 可生成一个通常由 SurfaceFlinger 使用的缓冲区队列，当渲染到 Surface 上时，结果最终将出现在传送给消费者的缓冲区中。也就是说，通过 Surface，数据将能进入缓冲队列了。

Surface 的创建和销毁需要时间，因而提供了 SurfaceHolder 回调，通知外界 Surface 的生命周期。同时 SurfaceHolder 也用来设置 Surface 的大小等属性。

#### 常见渲染组件

在对图形架构有些大体上的认识后，我们也简单了解下 Android 常见的渲染组件是怎么回事。

1. SurfaceView
普通的 View 在绘制的时候，都会通过会渲染到一个由 SurfaceFlinger 创建好的 Surface 中去。SurfaceView 与 普通 View 不同的是，会要求 SurfaceFlinger 创建一个新的 Surface，注意是 `新` 的。而在真正绘制的时候，会通过 Z-Order 将这个新的 Surface 放置在顶层，而被遮挡的 View 部分变为透明。最后 SurfaceFlinger 会合并多个 Surface，从而得到想要的效果。也正因为如此，SurfaceView 的绘制周期可能与 UI 线程上略有差异，这也是 SurfaceView 不好的地方。

2. GLSurfaceView
前文提到 OpenGL ES，我们在 UGC 开发的时候，需要用到 OpenGL ES。Android 端对应的 OpenGL ES 实现为 EGL，在 Android 端开发的时候，使用 EGL 即可操作 OpenGL API。GLSurfaceView 帮我们封装了 EGL 相关内容，我们不需要创建额外的渲染线程，也不用配置 EGL Context。然而在 onDrawFrame 的时候，我们又需要调用 OpenGL 相关 API 来进行绘制，从而显示出画面。GLSurfaceView 对 EGL 进行了不彻底的封装，有时候会显得很尴尬，对 GLSurfaceView 的使用，可自行抉择。

3. SurfaceTexture
> Captures frames from an image stream as an OpenGL ES texture.

SurfaceView 可以理解为 Surface + View 的组合，SurfaceTexture 也可以认为是 Surface + Texture 的组合，其中 Texture 为 OpenGL ES 的纹理。SurfaceTexture 可以不断地从 BufferQueue 中获取数据，并输出纹理(GL_TEXTURE_EXTERNAL_OES或者GL_TEXTURE_2D)。这个纹理，又能被外界的 GLES 所使用。下图是对 SurfaceTexture 在相机预览中使用的例子。

![Camera Preview SurfaceTexture](http://o8p68x17d.bkt.clouddn.com/3.png)

4. TextureView
TextureView 是 View 层次结构的固有成员，使用 TextureView 时，View 合成往往使用 GLES 执行，并且对其内容进行的更新也可能会导致其他 View 元素重绘，这相比 SurfaceView 有好处也有坏处。

> Unlike SurfaceView, TextureView does not create a separate window but behaves as a regular View. This key difference allows a TextureView to be moved, transformed, animated, etc. For instance, you can make a TextureView semi-translucent by calling myView.setAlpha(0.5f).

#### 参考资料

1. [Android硬件加速原理与实现简介](https://tech.meituan.com/hardware-accelerate.html)
2. [图形架构](https://source.android.google.cn/devices/graphics/architecture)

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2018年4月12日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
