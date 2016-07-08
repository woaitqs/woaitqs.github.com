---
layout: post
title: "Android 开发者选项详述"
description: "如何用好 Android 开发者选项，为你的开发做贡献"
keywords: "android, 开发者选项, developer options, OverDraw"
category: "android"
tags: [android, tools]
---
{% include JB/setup %}

<div style="border:solid 1.5px #ccc;padding:20px 20px 10px 20px;margin-bottom: 20px;border-radius: 6px;">

          <span style="position: relative;padding: 5px 15px 5px 15px;background-color: white;bottom: 30px;font-size: 16px;font-weight: bolder;">摘要</span>

          <span style="position: relative;display: block;bottom: 15px;">
          本文列举了常用的 Android 开发者选项，了解和熟练使用这些开发者选项，能够帮助我们定位开发中遇到的问题，辅助我们了解应用的性能问题，对提升开发和优化效率大有帮助。
          </span>
</div>

#### 1. Stay awake (不锁定屏幕)

使用场景：在使用 USB 进行调试的时候，经常调试一段时间后，想在手机上进行下一步操作，突然发现手机就黑屏，需要解锁。

使用说明：调试时屏幕一直常亮，妈妈再也不用担心调试的时候，黑屏啦！

<!--break-->

#### 2. Process Stats (进程统计信息)

![process stats](http://o8p68x17d.bkt.clouddn.com/process_stats.png)

使用场景: 查看后台进程和资源占用，以图形的方式展示了后台运行的进程，以及相应的运行时间和内存占用。

使用说明: 如图，左上角是指其统计的时间范围，而其下面的条形区域的进度颜色则显示了当前内存使用的情况，绿色表示处于正常范围，黄色则表示有些紧张，红色则是告急状态。再下面的列表区域则显示了当前运行的进程，右上方的百分比标明其在这段时间内运行的相对时间，100% 就表示其在这段时间内都在运行。点击进入，能够看到起内存占用详细信息。

![process detail](http://o8p68x17d.bkt.clouddn.com/process_detail.png)

在图中，分别显示了内存(RAM)占用情况，以及运行的 Services 列表。

这些信息也可以通过 adb 来查看，相应的命令如下:

> adb shell dumpsys activity (ActivityManager 系统服务的相关信息，这些信息包括 Activity，Broadcast，Service 和 ContentProvider)

> adb shell dumpsys meminfo （内存使用信息）

> adb shell dumpsys procstats --hours 3 (查看过去 3 小时内，进程的使用情况)

更多信息参考 [链接](http://android-developers.blogspot.hk/2014/01/process-stats-understanding-how-your.html)

#### 3. Wait for debugger & Select debug app （等待调试器 和 选择调试应用）

使用场景: 遇到一些需要开启 APP 急需 Debug 的情况，或者需要调试 APP 启动崩溃时。这时候通常来不急挂载断点，App 进程就崩溃了。

使用说明: 在 `Select debug app` 选择开发 APP，并勾选 `Wait for debugger`, 然后再启动应用。

![wait for debugger](http://o8p68x17d.bkt.clouddn.com/wait_for_debugger.png)

#### 4. Show touches & Pointer location (显示触摸操作 和 指针位置)

使用场景: 在查看 view 点击区域，或者查看触摸手势时，需要对点击位置和操作进行相应的查看。

使用说明:  `show touches` 显示了触摸位置，`Pointer location` 则显示了触摸手势。

#### 5. Animation scale (动画程序时长缩放)

使用场景: 调试复杂动画，可以放慢动画效果，以便仔细观察和调试动画。

使用说明: 开启后，选择相应的缩放比，就能明显感知。

#### 6. Show layout bounds (显示布局边界)

使用场景: 查看 view 的区域，以及相应的 margin 和 padding.

使用说明: 开启后就能看到效果.

![layout bounds](http://o8p68x17d.bkt.clouddn.com/layout_bounds.png)

#### 7. Debug GPU OverDraw (调试 GPU 过度绘制)

先来看看什么是过度绘制。我们在绘制界面的时候，往往会有多个层级，例如在一块白色背景上绘制了一张图片，但图片下面遮住的白色背景是我们所看不到的，这一部分也是不需要绘制的，我们称这种现象为 过度绘制。显然，过度绘制造成了额外的工作，是我们应该尽可能地避免的问题。

![over draw](http://o8p68x17d.bkt.clouddn.com/overdraw.png)

使用场景: 查看开发的 APP 是否存在很严重的过度绘制问题。

使用说明: 开启后就能看到效果，选择 `Debug GPU OverDraw`, 并勾选 `Show overdraw areas`。过度绘制根据额外绘制的层级数，分为蓝(1x)，黄(2x), 红(3x), 深红(4x+), 应该尽可能地使得我们的界面层级保持在蓝色或者黄色。

#### 8. Profile GPU rendering（GPU 呈现模式分析）

![Profile GPU  Rendering](http://o8p68x17d.bkt.clouddn.com/profile_gpu_rending.png)

使用场景: 如我们所知，如果一阵的绘制时间超过了 16 ms，那么用户就能实际地感受到视觉上的差异，这也就是我们常说的卡顿。GPU 呈现模式能使得我们以图形化的方式查看绘制每一帧花费的时间，以及其是否超过 16 ms，在这种模式下，可以比较粗略地定位在那一块操作比较卡顿。我们分析下图片，图片中有很多竖着的线，这些竖着的线表示一帧，其中竖线的每个颜色都表示着这一阵在绘制中的某个步骤，高度就是其花费的时间。上方的这个横线，表示16ms，任何一根竖着的线都可以和 16ms 进行比较，如果其超过 16ms，那么它的绘制时间就超过了建议的时间范围，会造成界面卡顿。开发者可以通过查看进行什么操作会使得竖线高度飙升，来初步定位卡顿问题。

使用说明: 点击 `Profile GPU rendering`, 选择 `On screen as bars`.

#### 9. Don't keep activities (不保留活动)

使用场景: 在实际的生产环境往往会触发一些比 Debug 环境更为严苛的问题，这里通常用来模拟内存受限，不可见 Activity 被回收的情况。在这种模式下，容易触发一些不常见的崩溃，便于开发者提升应用的稳定性。

使用说明: 开启 `Don't keep activities` 即可。

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年7月1日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
