---
layout: post
title: "Binder -- 稳定的传送门"
description: ""
category: "android"
tags: [program]
---
{% include JB/setup %}

如果各位玩过《炉石传说》，那么可能对法师的职业卡「不稳定的传送门」很有印象，特别是没有欧洲玩家，经常能够拿到其他职业的强力单卡。Android 也提供了传送门，让我们可以像使用本地方法一样，调用其他进程的方法，他有一个响亮的名字，Binder！

<!--break-->

--------------

### 什么是Binder? 为什么我们需要它？

在提及Binder之前，我们先来看看Android的设计。在Linux系统里面，进程之间是相互隔离的，也就是说进程之间的各个数据是互相独立，互不影响，而如果一个进程崩溃了，也不会影响到另一个进程。这样的前提下将互相不影响的系统功能分拆到不同的进程里面去，有助于提升系统的稳定性，毕竟我们都不想自己的应用进程崩溃会导致整个手机系统的崩溃。而Android是基于Linux系统进行开发的，也充分利用的进程隔离这一特性。

这些Android的系统进程，从System Server 到 SurfaceFlinger，qcks，各个进程各司其职，支撑起整个Android系统。而我们进行的Android开发也是和这些系统进程打交道，通过他们提供的服务，架构起我们的App程序。那么有了这些进程之后，问题紧接着而来，我们怎么和这些进程合作了？答案就是`IPC`(进程间通信)。

Linux System 在IPC中，做了很多工作，提供了不少进程间通信的方式，下面罗列了几种比较常见的方式。

- Signals 信号量
- Pipes 管道
- Socket 套接字
- Message Queue 消息队列
- Shared Memory 共享内存

按照复用的角度上看，既然有这么多"轮子"后，就应该合理利用这些"轮子"，从而方便地调用系统服务。然而事实并没有这么简单，Android系统作为嵌入式的移动操作系统，通信条件相对更加苛责一些，苛责的地方提现在：

- 拮据的内存，移动设备上的内存情况不同于PC平台，内存受限，因而需要有合适的机制来保证对空闲进程的回收
- Android 不支持System V IPCs
- 安全性问题显得更为突出，移动平台特有的权限问题
- 需要Death Notification（进程终止的通知）的支持

由于前面提及的特殊性，先前的轮子已经不能满足所有的需求了，因而就有了今天的主角 Binder。
Binder 是一个基于OpenBinder开发，Google在其中进行了相应的改造和优化，在面向对象系统里面的IPC/组件，适配了相关特性，并致力于建立具有扩展性、稳定、灵活的系统。

在这一小节结束的时候，来看一看Binder在Android系统中的使用场景，也就是图中的IPC模块。

![Binder 存在的地方](http://i2.buimg.com/f6eaf162ea1a609f.png)

--------------

### Binder 概述

既然需要重新造轮子，那么接下来让我们沿着设计者的思路，来看看如何一步一步满足前面提及的特殊需求。Binder在本质上需要解决的问题是让两个不同的进程之间能够互相调用的问题，所以从开发者的视角来看，应该就简单地如下图：

![binder-user.png](https://ooo.0o0.ooo/2016/05/24/57451b2f09a3b.png)

--------------

### 参考文献

1. [http://mindtherobot.com/blog/159/android-guts-intro-to-loopers-and-handlers/](http://mindtherobot.com/blog/159/android-guts-intro-to-loopers-and-handlers/)

--------------