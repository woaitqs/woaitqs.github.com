---
title: Android UGC Client Tutorial II
date: 2018-04-12 18:51:08
layout: post
description: "如何搭建 UGC 客户端, 第二部分，构建一个最简播放器"
category: "android"
tags: [ugc, android, Tutorial, arch, surface]
---

前文学习了 Android 图形架构的理论知识，现在来用一个实际的例子加深我们的理解。做一个超简单的视频播放功能，在这个例子中可以学习到如下几个知识：

1. Android 视频编解码

2. Surface 使用

3. OpenGL 相关的使用


------------------

### 视频基础知识

通常视频文件是有后缀名，例如 mp4 , flv 等等，这些内容被称为封装格式。

> 视频封装格式，简称视频格式，相当于一种储存视频信息的容器，它里面包含了封装视频文件所需要的视频信息、音频信息和相关的配置信息(比如：视频和音频的关联信息、如何解码等等)。

对应的也有视频格式和音频格式，这里介绍最通用的视频格式和音频格式。

##### 视频格式 -> H.264

H.264，等同于 MPEG-4 第十部分，也被称为高级视频编码(Advanced Video Coding，简称 AVC)，是一种视频压缩标准，一种被广泛使用的高精度视频的录制、压缩和发布格式。该标准引入了一系列新的能够大大提高压缩性能的技术，并能够同时在高码率端和低码率端大大超越以前的诸标准。

##### 音频格式 -> AAC

AAC，英文全称 Advanced Audio Coding，是由 Fraunhofer IIS、杜比实验室、AT&T、Sony等公司共同开发，在 1997 年推出的基于 MPEG-2 的音频编码技术。2000 年，MPEG-4 标准出现后，AAC 重新集成了其特性，加入了 SBR 技术和 PS 技术，为了区别于传统的 MPEG-2 AAC 又称为 MPEG-4 AAC。

##### 音频格式 -> MP3

英文全称 MPEG-1 or MPEG-2 Audio Layer III，是当曾经非常流行的一种数字音频编码和有损压缩格式，它被设计来大幅降低音频数据量。它是在 1991 年，由位于德国埃尔朗根的研究组织 Fraunhofer-Gesellschaft 的一组工程师发明和标准化的。MP3 的普及，曾对音乐产业造成极大的冲击与影响。

更多知识参考[关于视频的一些概念](http://www.samirchen.com/video-concept/)

------------------

### Android 编解码

如果要实现一个视频的播放功能，那么就需要了解这个视频的格式和其他相关信息。


##### 视频信息获取

MediaExtractor 可以实现这样的功能，API 也相对简单。视频是有多个轨道的，一到多条音轨，一个视频轨，还可能有字幕轨等等，下面的代码，就是在选择视频轨。


```java

extractor = new MediaExtractor();

extractor.setDataSource(mSourceFile.toString());

int trackIndex = selectTrack(extractor);

if (trackIndex < 0) {

    throw new RuntimeException("No video track found in " + mSourceFile);

}

extractor.selectTrack(trackIndex);

MediaFormat format = extractor.getTrackFormat(trackIndex);

```


##### 视频编解码


视频是有多种格式的，每种各式的编解码方式各不一样，大多程序都在使用 FFmpeg 做类似的工作，现在也有部分客户端使用硬编硬解的方式。MediaCodeC 就是 Android 引入的 API，用来进行编解码的。





在使用 MediaCodeC 之前，需要先对其进行配置，这样 MediaCodeC 才能知道是什么样的视频格式。



![MediaCodeC #config](http://o8p68x17d.bkt.clouddn.com/4_18_1.png)



MediaCodeC 进行编解码的工作，非常适用于生产者消费者模式，实际上也是这么实现的，这样编码和解码工作就能互相解耦了。MediaCodeC 可以操作三种数据：



1. Compressed Data

2. Raw Video Data

3. Raw Audio Data



例如我们获取到了原始的 Audio Data，就可以将其交给 MediaCodeC 进行相应的解码成 ByteBuffer。也可以将原始的 PCM (ByteBuffer) 编码成 AAC 格式。



这里需要特殊说明的是，对于 Video 格式的数据，可以使用 Surface 来增强性能。Surface 使用native buffer，没有经过映射、拷贝到 ByteBuffers，所以效率更高。正常情况下，使用 Surface 时获取不到 raw video data，需要通过ImageReader类获取raw格式的vedio帧数据，由于native层的buffers可以直接映射为ByteBuffer，所以效率依然很高。当使用ByteBuffer模式时，通过Image类的getInput()/OutPutImage(int)方法，就可以得到raw格式的video帧数据。



------------------



### 极简视频播放功能实现



1. 首先确定视频的格式，选择其中的视频轨道，创建相应的 MediaCodeC Decoder。



![MediaCodeC Decoder](http://o8p68x17d.bkt.clouddn.com/4_18_2.png)



需要额外指出的是，MediaCodeC 在 Config 的时候，指定了 Surface 作为输出格式，也就是说解码出来的内容直接输出到 Surface 上。



2. 然后轮询解码视频，输出到 Surface 上。

![轮询](http://o8p68x17d.bkt.clouddn.com/4_18_3.png)

在调用 releaseBuffer 的时候，MediaCodec 会自动将内容输出到 Surface。



3. 将 Surface 和 Texture 中的 SurfaceTexture 绑定起来，这样 TextureView 就能不断显示 MediaCodec 解码出的视频帧了。

![bind each other](http://o8p68x17d.bkt.clouddn.com/4_18_4.png)

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2018年4月18日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
