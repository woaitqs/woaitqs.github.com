---
layout: post
title: "Android Binder 全解析(1) -- 概述"
description: "对Android Binder的基本描述"
keywords: "Android, Binder, 机制, 实现原理"
category: "android"
tags: [android, program, binder]
---
{% include JB/setup %}

<div style="border:solid 1.5px #ccc;padding:20px 20px 10px 20px;margin-bottom: 20px;border-radius: 6px;">

          <span style="position: relative;padding: 5px 15px 5px 15px;background-color: white;bottom: 30px;font-size: 16px;font-weight: bolder;">摘要</span>

          <span style="position: relative;display: block;bottom: 15px;">
          如果各位玩过《炉石传说》，那么可能对法师的职业卡「不稳定的传送门」很有印象，特别是没有欧洲玩家，经常能够拿到其他职业的强力单卡。Android 也提供了传送门，让我们可以像使用本地方法一样，调用其他进程的方法，他有一个响亮的名字，Binder！

          Binder 在 Android 是如此的重要，承当起整个Android的通信任务，作为优秀的Android工程师有什么理由不了解了？在接下来的文章中，会陆陆续续讲解Android Binder，希望大家持续关注。

          本文是 Android 系统学习系列文章中的第二章节，在前面一些细节概念的铺垫下，大体上知道 Binder Framework 是怎么运作的，在这篇文章中，将详细说明下 Binder Framework 的具体实现，这一套机制如何盘活整个 Android 系统。对此系列感兴趣的同学，可以收藏这个链接 <a herf="http://www.woaitqs.cc/2016/06/16/android-system-learning.html">Android 系统学习</a>，也可以点击 <a href="http://www.woaitqs.cc/feed.xml">RSS订阅</a> 进行订阅。
          </span>
</div>

- **DONE:** [Android Binder 完全解析（一）概述](http://www.woaitqs.cc/android/2016/05/23/android-binder.html)
- **DONE:** [Android Binder 完全解析（二）设计详解](http://www.woaitqs.cc/android/2016/05/26/android-binder-token.html)
- **DONE:** [Android Binder 完全解析（三）AIDL实现原理分析](http://www.woaitqs.cc/android/2016/05/30/android-binder-proxy-and-token.html)

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

### Binder Framework 愿景

既然需要重新造轮子，那么接下来让我们沿着设计者的思路，来看看如何一步一步满足前面提及的特殊需求。Binder在本质上需要解决的问题是让两个不同的进程之间能够互相调用的问题，所以从开发者的视角来看，应该就简单地如下图：

![binder-user.png](http://o8p68x17d.bkt.clouddn.com/binder-user.png)

同时从效能的角度上出发，希望提供服务调用的程序能够支持并发，这样使得能够尽可能地响应多个程序的IPC请求，由此得出的实际运行图是下面这个样子的。

![biner-multi-thread.png](http://o8p68x17d.bkt.clouddn.com/biner-multi-thread.png)

--------------

### Binder Framework 实现细节

有了前面这些愿景后，再来看看Binder Framework的一些实现细节，如何走到这一步的，当然这是非常泛的细节，作为常识了解一下。

#### 如何跨进程调用

那么如何使得调用者能像上述一样简单地调用远程方法？毕竟两者存在于不同的进程空间里面。那么可以引入一个黑盒模块，用这个黑盒模块来帮我们完成其中的细节，这个模块也被称为Binder Driver。方法的跨进程调用受到了 Linux 进程隔离的限制，而解决方案就是将其置于所有进程都能共享的区域 -- Kernel，而 Binder Driver 提供的功能也就是让各进程使用内核空间，将进程中的地址和Kernel中的地址映射起来，其中Linux ioctl 函数实现了从用户空间转移到内核空间的功能。在 Binder Driver 的支持下，就能实现跨进程调用。

#### Client / Server 架构

在设计的时候，Binder Framework 交互模型采用的是客户端/服务器模型。客户端需要调用远程服务的内容时， 会初始化一个连接，并等待服务器的返回，同时会block住自己。Binder Framework在客户端这边实现了一个代理，而在服务端，通过线程池的方式来响应请求。在如下图所示，A进程就是客户端，并且通过Proxy来完成对Binder Driver内核的交互。Process B是系统服务进程，在这个进程里面维护着多个Binder Thread，直到达到设置的线程上限。Proxy对象通过和Binder Driver进行交互，从而使得Binder Driver将信息传递到目标对象。从Android开发者的角度出发，Binder Framework提供的最方便的改进就是能像调用本地方法一样调用远程方法或对象。客户端的进程调用会在Server进程返回之前一直处于block的状况。在这种机制下，客户端就不必提供一个单独的线程模型和回调机制。（同步转异步简单，而异步变同步则很困难）

![binder-proxy.png](http://o8p68x17d.bkt.clouddn.com/binder-proxy.png)

#### 传递的数据格式

在实现跨进程调用的时候，涉及到参数和命令的传递，得有一个合适的数据结构来表达需要远程执行的东西。
![binder-data.png](http://o8p68x17d.bkt.clouddn.com/binder-data.png)

Target是指目标binder，Cookie这涵盖着一些内部信息，sender Id则包含了安全相关的信息，data则包含着一些数据的数组。每个数组的Entry是由相关的命令和参数组成的，这部分参数将传递给目标binder。

而这里面的Sender Id 则非常的重要，不仅可以起到唯一标示Binder的作用，还可以在跨进程的地方作为标记的作用，在接下来的文章里再详细说明。

### Service Manager

我们接触的服务很多，从Display到Location，从Audio到Wifi，如果我们和每一个服务都通过前面描述的方式进行交互，即便通过 Proxy 的方式也是非常的繁琐。而且在调用每个系统服务的时候，必须知道对应的系统服务的地址，而系统服务的地址出于安全性的考虑也不应该暴露出来。那么如何方便我们进行系统服务调用了？

Service Manager 就是来帮助我们解决这个问题。这是Binder Framework的一个特殊节点，也是第一个起点。其作为一个命令服务，起到了DNS的作用，使得可以通过名字的方式来查找相应的Binder接口。这很重要，因为客户端不应该知道远程服务的调用地址，如果知道了这势必会很不安全。每一个Binder需要将自己的名字和Binder Token交给Service Manager，客户端只需要知道服务的名字就可以。

![binder-mananger.png](http://o8p68x17d.bkt.clouddn.com/binder-mananger.png)

### 参考文献

1. [http://mindtherobot.com/blog/159/android-guts-intro-to-loopers-and-handlers/](http://mindtherobot.com/blog/159/android-guts-intro-to-loopers-and-handlers/)
2. [http://www.androiddesignpatterns.com/2013/08/binders-death-recipients.html](http://www.androiddesignpatterns.com/2013/08/binders-death-recipients.html)
3. [http://events.linuxfoundation.org/images/stories/slides/abs2013_gargentas.pdf](http://events.linuxfoundation.org/images/stories/slides/abs2013_gargentas.pdf)

4. http://www.androiddesignpatterns.com/2013/08/binders-death-recipients.html
5. http://www.androiddesignpatterns.com/2013/07/binders-window-tokens.html


--------------
