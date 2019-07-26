---
title: Android 硬编硬解退坑指南
date: 2018-09-01 16:52:50
layout: post
description: "这篇文章将重点介绍了，硬编硬解过程中遇到的坑，也给读者朋友架构方案时提供一点参考"
category: "android"
tags: [ugc, android, Tutorial, mediacodec, camera]
---

抖音、快手在国内迅速走红，也带动了国内短视频的热潮。短视频录制、编辑等等功能，是一项系统性、专业性很强的领域。经过一段时间发展后，有多种方式可以通向罗马，但并不是每一条路都好走。这篇文章将重点介绍下，硬编硬解过程中遇到的坑，也给读者朋友架构方案时提供一点参考。

------------------

### Camera 使用的坑
虽然和硬编硬解没有关系，但短视频客户端不可能绕过这个，这里也列举下我所遇到的问题。

#### API 使用上的一些坑

Camera API 是 Android 碎片化严重的体现，不同于 iOS 系列，一共就那几种机型。Android 厂商众多，ROM 也百花齐放，同样的 API 在不同机型上提现可能完全不同，也是我们需要在开发过程中留意的。下面简单列举两个可能会导致画面异常的 API。

1. setRecordingHint

> Sets recording mode hint. This tells the camera that the intent of the application is to record videos MediaRecorder.start(), not to take still pictures Camera.takePicture(Camera.ShutterCallback, Camera.PictureCallback, Camera.PictureCallback, Camera.PictureCallback). Using this hint can allow MediaRecorder.start() to start faster or with fewer glitches on output. This should be called before starting preview for the best result, but can be changed while the preview is active. The default value is false. The app can still call takePicture() when the hint is true or call MediaRecorder.start() when the hint is false. But the performance may be worse.  

官方的注释如上，英文很简单，就不翻译了。看上去是一个很人畜无害的 API，但实际上却暗藏杀机。此方法需要你设置 video-size，如果不设置这个，将会在某些机型(OPPO A37f, MI 2 , LG 部分手机)上造成拉伸的后果。
![图片来源于网络](https://i.loli.net/2018/11/13/5beaa5d3dee99.png)


RecordingHint 的目的是帮助更快速地进行录制，某些 ROM 在实现的时候，没有考虑到 Preview Size  和 Video Size 之间的区别，例如使用 480 * 864 的分辨率进行录制，输出的文件大小为 400 * 800，那么势必会使得整体画面出现拉伸的情况。

2. setVideoStabilization

> Enables and disables video stabilization. Use isVideoStabilizationSupported() to determine if calling this method is valid. Video stabilization reduces the shaking due to the motion of the camera in both the preview stream and in recorded videos, including data received from the preview callback. It does not reduce motion blur in images captured with takePicture. Video stabilization can be enabled and disabled while preview or recording is active, but toggling it may cause a jump in the video stream that may be undesirable in a recorded video.  

官方文档上写着，这个方法可以帮助提高录制视频的稳定性。使用的时候通过 `#isVideoStabilizationSupported` 来判断是否可以使用。如果妄自使用的话，会造成页面黑屏，PreviewCallback 没有任何回调。这两个 API 都是官方文档中提及，但完全没有任何隐患说明的地方，Camera API  调用的坑可见一斑。

#### Camera 效果不一致

如果 UI 妹子对画面要求比较高，需要在各大手机品牌上取得一致效果，那对于 Android Developer 简直是一场灾难。例如同样的白平衡效果，在 google 旗舰机型 Pixel 2XL 与 Huawei Honor 的差异就很明显。一般对于 Android 端，我们不能强求一致的效果，而是要提供给用户足够的自由度，让他们在机器提供的能力范围内拍出较为满意的效果。

Oppo 的部分机型自带美颜效果，而其他较为老旧的机型则没有这样的功能，因而我们对于这些机型上的默认美颜处理可以有不同的策略。

并不是所有机型都对拍出的照片添加了 Exif 信息，所以我们不能默认照片都是竖直方向的。如果照片中含有 Exif，还是要根据这个来判断照片方向。

总结起来说，我们在使用 Android Camera API 时，需要对曝光、色温、快门这些基本的拍摄概念有一个了解，这样才能在遇到各种奇怪情况的时候找到合适的解决方案。

------------------

### 硬编硬解的若干坑

#### MediaCodec 不一定支持
首当其冲的就是 MediaCodec 不一定支持！特别是一些老旧机型比较突出，在引入 CTS 测试之前，那简直了（手动微笑）。建议在合适的地方，判断硬编硬解支持的程度。

这里列举出 CTS 中测试 MediaCodec 支持程度的链接，[tests/tests/media/src/android/media/cts/EncodeDecodeTest.java - platform/cts - Git at Google](https://android.googlesource.com/platform/cts/+/kitkat-release/tests/tests/media/src/android/media/cts/EncodeDecodeTest.java)，大家可以参考这其中的代码来判断用户机型的支持程度。 ~~如果机型不支持，大概率在创建 Encoder 的时候就会崩溃。~~

#### Surface 也不一定支持
MediaCodec 支持的数据类型中，有一项就是 Surface。Surface 直接使用 native 层的 Buffer，这样避免映射和拷贝，效率上能高不少。然而并不是每一种机型都支持 Surface 格式的。

![create inputsurface](https://i.loli.net/2018/11/13/5beaa5fe65691.png)


我们可以在 createInputSurface 的时候 catch 下异常，如果代码上没有问题，那真的就是机型上不支持 Surface 了，那只能老老实实用 ByteBuffer 吧(手动扇子脸)。

#### 虚晃一枪的 API

备注：笔者使用的是 Google Pixel 2XL (Version 8.1) 进行地测试。

编解码本身灵活性就相对欠缺，可配置的地方就少，但就算在这种情况下那些看上去不错的接口，很大概率上只是镜中月，水中花，梦里人。

1. BitMode
文档上写着三种支持三种码率模式，CQ的意思是尽量保证帧率，保证图片质量；CBR 则是尽量保证每一帧的码率，但在一些动作比较大的Video中，就很容易模糊；VBR 则是前两者之间一个中庸的方案。
但是，绝大多数机型都只支持 VRB 一种。（如果有错误，欢迎指正）
![bit mode](https://i.loli.net/2018/11/13/5beaa61d9e7cd.png)


2. CodecLevel、CodecProfile
设置这两个值，可以改变编码级别，不同编码级别得出的效果不一样，级别越高，效果越好。我们可以通过 ffmpeg，或者 mediaInfo 来查看视频的编码级别。笔者在一些7.0＋的手机设备，并且支持设置level、profile 中，分别进行测试，发现所有视频编码出的效果，都是 `Base Media/ Version 2`。又一个并没有什么卵用的 API，笔者只能认为这是 Google 为日后埋下的伏笔了。同理 getComplexityRange 这个 API，可以有 low、upper两种级别，但它们的值都是0，可能 low 和 upper 是双胞胎吧（手动扇子脸）。即便有部分机型，可以设置 Profile 和 level并且生效，但如果你没有 B 帧的需求，强行设置 Profile 和 Level 的话，可能会造成 pts 和 dts 两者的差异，从而视频播放卡顿。

![codec profile](https://i.loli.net/2018/11/13/5beaa637ef869.png)


#### 隐式限制很多

MediaCodec 在 Configure 的过程中，可以设置各种各样的颜色格式。但实际上支持的种类非常有限，调用前一定要通过 `getCapabilitiesForType`来确认下。

一般来说，绝大多数相机的 Preview 输出都支持 NV21，因此我们在使用 Camera API 的时候，可以放心地在 Camera#Parameters setPreviewFormat 使用 NV21 格式。然鹅，MediaCodec 不一定支持 NV21 哦！这些支持的格式里面，不同厂商表现不一样哦，高通支持的别家还真不一定支持。底层在使用的时候，一定要做好判断。忘了说啦， `getCapabilitiesForType`这个 API 也要处理好异常（手动扇子脸 X3）。

写到这里，还需要额外指出一点，16位对齐的事情。软编软解的时候，可以放心使用 540 * 960 这样的分辨率，但在硬编硬解的时候，一定要用 544 * 960，如果没有16位对齐，才不说什么 `MOTO 机型直接花屏给你看` 的事情。

------------------

### 结语

出了上述的问题以外，其他 API 的限制(坑)都不少，例如 MediaMuxer 基本上和 MPEG-4 绑定死了。那这么说起来，硬编硬解在 Android 上是不是就不能用呢。其实也没那么不堪，在编解码速度上比软编软解快上不少，笔者简单进行了下对比，能快上50%(不具有参考性，机型越好，差距越少)。这里只是写一篇文章，分享下在实际开发中遇到的坑，给各位读者大大一点参考。

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2018年9月1日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------