---
layout: post
title: "Android Binder 全解析(2) -- 设计详解"
description: "Binder Framework 是如何设计实现的"
keywords: "Android, Binder, 机制, 实现原理"
category: "android"
tags: [android, program, binder]
---
{% include JB/setup %}

<div style="border:solid 1.5px #ccc;padding:20px 20px 10px 20px;margin-bottom: 20px;border-radius: 6px;">

          <span style="position: relative;padding: 5px 15px 5px 15px;background-color: white;bottom: 30px;font-size: 16px;font-weight: bolder;">摘要</span>

          <span style="position: relative;display: block;bottom: 15px;">
          在上一篇文章中介绍了什么是Binder? 为什么我们需要它？在这一篇文章中，将通过类比的思路来介绍 Binder 的设计原理，作为上一篇文章的补充。这篇文章只是从设计的概念出发进行理解，不设计太多的代码细节，如果想对具体实现感兴趣，可以参考老罗的文章。希望通过这篇文章，能够帮助大家理解整个 Binder 运作机制。
          在这篇文章结束后，将介绍 Binder Token的 妙用，和 Death Notification 的使用场景，感谢大家持续关注。
          本文是 Android 系统学习系列文章中的第二章节，在前面一些细节概念的铺垫下，大体上知道 Binder Framework 是怎么运作的，在这边文章中，将详细说明下 Binder Framework 的具体实现，这一套机制如何盘活整个 Android 系统。对此系列感兴趣的同学，可以收藏这个链接 <a herf="http://www.woaitqs.cc/2016/06/16/android-system-learning.html">Android 系统学习</a>，也可以点击 <a href="http://www.woaitqs.cc/feed.xml">RSS订阅</a> 进行订阅。
          </span>
</div>

- **DONE:** [Android Binder 完全解析（一）概述](http://www.woaitqs.cc/android/2016/05/23/android-binder.html)
- **DONE:** [Android Binder 完全解析（二）设计详解](http://www.woaitqs.cc/android/2016/05/26/android-binder-token.html)
- **DONE:** [Android Binder 完全解析（三）AIDL实现原理分析](http://www.woaitqs.cc/android/2016/05/30/android-binder-proxy-and-token.html)

<!--break-->

### 我们是怎么上网的？

在互联的时代，将各种设备连接上互联网，是我们使用一个新设备的第一件事情，网络无处不在。在这个互联的社会，是如何标记彼此呢？如何保证我想要上京东的时候，打开的网页不是淘宝？保证每一个网络设备能够被独立标志的功臣就是 Internet Protocol address。 IP 地址就是用于在互联网通信过程中，分配给设备的唯一标示，其他设备可以通过这一唯一标示访问当前设备。下面显示的就是 google.com.hk 的 IP 地址。

```xml
$ ping www.google.com.hk
PING www.google.com.hk (74.125.203.94): 56 data bytes
64 bytes from 74.125.203.94: icmp_seq=0 ttl=43 time=53.749 ms
64 bytes from 74.125.203.94: icmp_seq=1 ttl=43 time=55.464 ms
64 bytes from 74.125.203.94: icmp_seq=2 ttl=43 time=56.658 ms
64 bytes from 74.125.203.94: icmp_seq=3 ttl=43 time=56.441 ms
64 bytes from 74.125.203.94: icmp_seq=4 ttl=43 time=55.051 ms
64 bytes from 74.125.203.94: icmp_seq=5 ttl=43 time=54.940 ms
```

当我们需要访问某个网站时，可以通过某个 IP 直接来访问，例如直接在浏览器上访问 [74.125.203.94](74.125.203.94) 就可以访问 Google，但这个 IP 地址就是一组数字，非常不利于记忆，所有 IP 地址都是数字，显然不方便我们使用。那么如何来解决这个问题？答案也是我们耳熟能详的 DNS（Domain Name System）。

域名服务器 提供的功能提供了域名翻译的问题，每个网站都可以有属于自己的域名，例如 [woaitqs.github.io](woaitqs.github.io) 、[mp.weixin.qq.com](mp.weixin.qq.com) 等等。通常我们只需要在服务端输入这个域名，就能访问到对应的 IP 地址。这个过程发生了什么？实际上域名解析服务器，给我们提供了翻译功能，也就是说域名服务器帮助我们维护着 域名 和 对应 IP 的Map。当我们输入一个域名时，服务器会自动将这个域名解析成对应的地址，并访问相应的服务，我们只需要记住相应的域名就可以。

使用域名服务器时，我们需要将我们的服务在域名提供商哪里注册进去，常见的域名提供商是 [万网](wanwang.aliyun.com)、[西部数码](http://www.west.cn/)，在注册过后，自定义域名的实际访问地址，就解决了通过域名访问我们服务的问题。

从域名到目的地并非简单的映射就可以完成，为了更好的扩展性和高效性，需要有完善的算法和设备支持，从这里可以看到[更详细介绍](https://zh.wikipedia.org/wiki/%E8%B7%AF%E7%94%B1)，将这个从源地址传输到目的地址的活动成为路由。

通常情况下，我们访问其他域名，就是将域名对应的地址作为 Server，而我们本身作为 Client，也就是我们常说的 Client\Server 架构。每一次访问相应网址的时候，都是 Client 向 Server 发送请求，并等待 Server 的回复。

### Binder 架构设计

前面粗略地介绍了 我们是怎么上网的？ 这个看上去和本文没有关系的内容，花那么多时间进行介绍，是为了方便大家理解 Binder 架构。两者之间存在着异曲同工之妙。

Binder 被设计出来是解决 Android IPC（进程间通信） 问题的，为什么选用 Binder，可以参看我的上一篇文章 [Android Binder 全解析（一）](http://woaitqs.github.io/android/2016/05/23/android-binder)。Binder 将两个进程间交互的理解为 Client 向 Server 进行通信，在接下来的内容中，会将两者结合起来进行类比。

先看一张图
![binder总体架构](http://img.my.csdn.net/uploads/201211/28/1354093025_3702.png)

如上图所示，Binder 架构分为 Client、Server、Service Manager 和 Binder Driver。

- Client: 服务调用者，一般就是我们应用开发者，通过调用诸如`List<PackageInfo> packs = getActivity().getPackageManager().getInstalledPackages(0);` 这样的代码，来向 ServerManager 请求 Package 服务。
- Server: 服务提供者，这里面会有许多我们常用的服务，例如 ActivityService 、 WindowMananger， 这些系统服务提供的功能，是的我们能够使用 Wifi，Display等等设备，从而完成我们的需求。
- Service Manager: 这里是类似于前文中的DNS，绝大多数的服务都是通过 Service Manager来获取，通过这个 DNS 来屏蔽掉 对其他Server的直接操作。
- Binder Driver: 底层的支持逻辑，在这里承担路由的工作，不论风雨，使命必达，即使对面的server挂掉了，也会给你相应的死亡通知单 (Death Notification)

总结起来说，应用程序(Client) 首先向 Service Manager 发送请求 WindowManager 的服务，Service Manager 查看已经注册在里面的服务的列表，找到相应的服务后，通过 Binder kernel 将其中的 Binder 对象返回给客户端，从而完成对服务的请求。

### Binder Driver 是怎样充当路由角色的？

对于有网络编程经验的人来说，[Socket](http://www.cnblogs.com/skynet/archive/2010/12/12/1903949.html) 是很常用的概念。在Linux系统中，一切都被认为是文件，网络流也是文件，同样 Socket 也是文件，遵循着 `open - write / read - close` 的模式，Binder Framework在设计的时候，也同样设计了类似的概念。

而在 Binder Framework 中 Binder 充当了 Socket 的角色，在不同的进程里面穿梭，提供了通信的基础。对Binder而言，Binder可以看成Server提供的实现某个特定服务的访问接入点， Client通过这个『地址』向Server发送请求来使用该服务；对Client而言，Binder可以看成是通向Server的管道入口，要想和某个Server通信首先必须建立这个管道并获得管道入口。我们知道如果要访问一个对象的话，需要拿到这个对象的引用地址，我们可以这么认为 Binder 就是远程对象的一个地址，通过这个 Binder 就能轻松地拿到远程对象的控制权，也可以说 Binder 是句柄，可能符合现在的场景。

而让 Binder 起到上诉神奇作用的就是 Binder Driver。Binder Driver 在这里的作用就是前面提及的路由器，它工作在内核态，通过一系列 `open()` , `mmap()`, `ioctl()` , `poll()` 等操作，指定了一系列的协议，实现了 Binder 在不同进程之间的传递工作，这里就不再详细阐述了，有兴趣的同学可以自行查看相关文档。

### Service Manager 怎么当DNS的？

根据前文的描述，Service Manager是将相应的服务名字转换成具体的引用，也就是说使得 client 能够通过 bidner 名字来从 Server 中拿到对 binder 实体的引用。这里唯一需要特别说明的地方在于，Service Manager 的特殊性。我们知道 Service Manager 是一个进程，其他 Server 也是另一个进程，他们之间是如何进行通信的了？在没有其他中间服务进程的参与下，Service Manager 与 其他进程如何凭空通信？

这就是先有鸡，还是先有蛋的问题。答案是先有鸡，也就是说 Service Manager 首先就被创建了，并被赋予了一个特殊的句柄，这个句柄就是 0 。换而言之，其他 Server 进程都可以通过这个 `0句柄` 与 Service Manager 进行通信，在整个系统启动时，其他 Server 进程都向这个 `0句柄` 进行注册，从而使得客户端进程在需要调用服务时，能够通过这个 Service Manager 查询到相应的服务进程。

![binder framework 工作原理](http://o8p68x17d.bkt.clouddn.com/binder_framework.jpg)


### Proxy 的由来

Binder Framework 作为一个底层框架，使用的场景相当的广，但也使得不太适合面向对象开发。为了满足这样的需求，Android 的工程师采用了代理模式 Proxy 来解决这个问题。

![代理模式 UML](http://o8p68x17d.bkt.clouddn.com/proxy_uml.png)

首先定义一套相同的接口，服务端 和 客户端分别使用这套接口，服务端具体实现了这套接口的相应逻辑，而客户端也实现了这套接口，不过接口里面的具体实现是调用相应的远程服务接口，将函数参数打包，通过Binder向Server发送申请并等待返回值。

做为Proxy设计模式的基础，首先定义一个抽象接口类封装Server所有功能，其中包含一系列纯虚函数留待Server和Proxy各自实现。由于这些函数需要跨进程调用，须为其一一编号，从而Server可以根据收到的编号决定调用哪个函数。而这里的 Binder 具有唯一的标示性，在后面的文章中，再来说明 Android 系统如何使用这一特性。


### 参考文献

1. [http://blog.csdn.net/universus/article/details/6211589](http://blog.csdn.net/universus/article/details/6211589)
2. [http://blog.csdn.net/luoshengyang/article/details/6618363](http://blog.csdn.net/luoshengyang/article/details/6618363)
3. [http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html](http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html)
