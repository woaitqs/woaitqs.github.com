---
layout: post
title: "Android Binder 全解析(2) -- 设计详解"
description: ""
category: "android"
tags: [program]
---
{% include JB/setup %}

在上一篇文章中介绍了什么是Binder? 为什么我们需要它？在这一篇文章中，将通过类比的思路来介绍 Binder 的设计原理，作为上一篇文章的补充。这篇文章只是从设计的概念出发进行理解，不设计太多的代码细节，如果想对具体实现感兴趣，可以参考老罗的文章。[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)。希望通过这篇文章，能够帮助大家理解整个 Binder 运作机制。

在这篇文章结束后，将介绍 Binder Token的 妙用，和 Death Notification 的使用场景，感谢大家持续关注。

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

通常情况下，我们访问其他域名，就是将域名对应的地址作为 Server，而我们本身作为 Client，也就是我们常说的 Client\Server 架构。每一次访问相应网址的时候，都是 Client 向 Server 发送请求，并等待 Server 的回复。

### Binder 架构设计

前面粗略地介绍了 我们是怎么上网的？ 这个看上去和本文没有关系的内容，花那么多时间进行介绍，是为了方便大家理解 Binder 架构。两者之间存在着异曲同工之妙。

Binder 被设计出来是解决 Android IPC（进程间通信） 问题的，为什么选用 Binder，可以参看我的上一篇文章 [Android Binder 全解析（一）](http://woaitqs.github.io/android/2016/05/23/android-binder)。