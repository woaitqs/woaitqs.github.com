---
layout: post
title: "详解 Android 是如何启动的"
description: "Android 从开机到运行 App，在这个过程中发生了什么？"
keywords: "android 启动 launch"
category: "android"
tags: [android, program, process]
---
{% include JB/setup %}

<div style="border:solid 1.5px #ccc;padding:20px 20px 10px 20px;margin-bottom: 20px;border-radius: 6px;">

          <span style="position: relative;padding: 5px 15px 5px 15px;background-color: white;bottom: 30px;font-size: 16px;font-weight: bolder;">摘要</span>

          <span style="position: relative;display: block;bottom: 15px;">
          本文是 Android 系统学习系列文章中的第一章节，从大体上说明 Android 系统是如何启动的？从开机到程序启动，发生了那些步骤，这些步骤意味着什么？欢迎进入今天的「走进科学」，逃 ：）。对此系列感兴趣的同学，可以收藏这个链接 <a herf="http://www.woaitqs.cc/2016/06/16/android-system-learning.html">Android 系统学习</a>，也可以点击 <a href="http://www.woaitqs.cc/feed.xml">RSS订阅</a> 进行订阅。
          </span>
</div>

<!--break-->

-------------------------

### 系统分区划分

Android 达人都经历过刷机的体验，如果通过 fastboot 来进行刷机的话，会在刷机界面看到如下的几个步骤。这些步骤是做什么用的？就是通过 fastboot 协议更新和烧录到 Android 手机对应的分区上。对 fastboot 感兴趣的同学，可以点击这个 [链接](https://android.googlesource.com/platform/system/core/+/master/fastboot/fastboot_protocol.txt) 进行查看。

```shell
fastboot flash boot boot.img
fastboot flash system system.img
fastboot flash userdata userdata.img
fastboot flash cache cache.img
fastboot flash recovery recovery.img
fastboot reboot
```

从 fastboot 的烧录过程中可以看出 Android 系统的大致分区，这里我也通过 adb shell 中的 df 命令查看了小米手机中的系统分区，如下图所示。

![小米系统分区](http://o8p68x17d.bkt.clouddn.com/xiaomi_file_system.png)

一般而言，虽然各个手机厂商在在系统上的实现不一致，Android 系统分为下面表格中的几个部分。

![Android 系统分区](http://o8p68x17d.bkt.clouddn.com/android-partitions.png)

分区 | 功能 |
------|-------
boot | 系统引导分区，包含着android 内核，系统没有这个无法启动。这一部分的镜像在 boot unlocked 时，也能够被擦除，但在这个过程中，不能被打断，关机等等，否者会导致系统无法启动。
system| Android 整个系统所在地，也包含预装的应用(这是预装的APP，手机厂商盈利的一种渠道)，
recovery | 备份分区，启动时可进入 recovery ，然后在这个模式下进行相应的 recovery 操作。
data | 应用程序相关的数据，例如你安装的豌豆荚，数据就放在 `/data/data/com.wandoujia.phoenix2` 下面，当进行恢复出厂设置时，这部分数据会被擦除掉。
cache | 用于存放缓存相关的数据。
misc | 存放一些硬件配置、USB 配置等等信息，如果被擦除可能会导致某些系统设备无法正常工作。

---------------------

### Android 系统启动

如我们所知，Android 系统是基于 Linux 系统开发而成，在其中针对移动设备的特性进行了相应的调整，所以 Android 系统大体上可以分为 Linux 内核部分和 应用系统部分。在 Android 系统启动的时候，也会先启动 Linux 内核，然后再启动应用系统。整个启动步骤分为 6 个部分，在下面进行详细的描述。

![Android 系统启动](http://en.miui.com/data/attachment/forum/201403/01/07115949n944zi559xt1zt.jpg.thumb.jpg)

#### 启动电源以及系统启动

当电源按下，引导芯片代码开始从预定义的地方（固化在ROM）开始执行。加载引导程序到RAM，然后执行。

#### Boot Loader

Android系统是基于Linux操作系统的，所以它最初的启动过程跟Linux一样。当设备通电时，首先执行BootLoader引导装载器，BootLoader是在操作系统内核运行之前运行的一小段程序。BootLoader 的概念在各个操作系统 windows, mac, Android 中都存在。通过这段小程序初始化硬件设置，建立内存空间映射图，从而将系统的软硬件环境导入到合适的状态，以使为最终调用操作系统内核准备好正确的运行环境。

BootLoader 在任何软件之前执行，因而也没有类似于虚拟机那样的概率来屏蔽不同运行环境的差异，因而某个主板都有特定的BootLoader。另一方面，iOS 系统由于其各个版本间的稳定性，因而不同版本件的 BootLoader 差异不大。另一方面，Android OS 是开源系统，其可以运行在不同的硬件上，这些硬件千差万别，因而在 Android 上的 BootLoader 也各有不同。

一般而言 BootLoader 出于锁闭状态，毕竟厂商开发出系统后，还是希望在 ROM 上的 Android 系统保持不变。因而刷机的第一步，往往是解锁 BootLoader.

总结起来，BootLoader 完成的工作是：
1. 初始化 RAM
2. 初始化基础硬件，例如 WIFI
3. 加载内核和内存空间映射图
4. 跳转到内核中

#### 内核模块
内核模块，负责了大部分的硬件、驱动和文件系统的初始化，这部分都在内核空间完成，不涉及用户空间的内容。内核模块的初始化是非常硬件相关的，目的就是使得 CPU 能够更快地执行 C 代码，然后在这执行完毕后，初始化各个子系统，来执行 `init` 进程。

#### Init 进程

Init 进程作为第一个执行的进程，开天辟地。这个进程主要做两件事情，挂在相应的目录(/sys,/dev 等等)，另一方面是执行 `init.rc` 脚本，可通过 [init.rc 源码](https://android.googlesource.com/platform/system/core/+/android-4.1.2_r1/rootdir/init.rc) 查看相应的链接。

Init 进程会启动 Runtime 运行时服务，所谓的 Runtime 服务就是将中间代码静态编译成本地代码。而 Android 所使用的 Java 动态执行，所依附的正式这个 Runtime 环境。

其后，Init 进程会启动一些本地守护进程，这些守护进程启动后，会初始化其相应的模块，在其中最特别的就是 Zygote。不过 Init 进程不会直接启动 Zygote 进程，而是使用 app_process 命令来通过 Android Runtime 来启动，Android Runtime 会启动第一个 Davlik 虚拟机，并调用 Zygote 的 main 方法。

#### Zygote and Dalvik

对于每一个运行的程序，Android 都会赋予其单独的虚拟机，以支持其运行，但每一次新建虚拟机的开销都不小，那么如何来缩短这个时间了？尤其是针对嵌入式设备，都希望相应速度能够达到理想的状况。Zyogte 就是用来解决这个问题的，其中文意思是受精卵。从这个名字上可以看出，其余的应用进程都是通过 fork 这个进程来实现的，他们共享相同的内存区域，这样能减少不少的内存占用开销和应用启动时间。

为了加速 App 的启动，Zygote 进程会预先加载 App 在运行时所需的资源和 Class 文件到系统 RAM 中。Zygote 会监听其 Socket (/dev/socket/zygote) 来判断是否需要启动 App。每当监听到需要创建 App 的请求时，就 fork 一个进程即可。这样的好处在于最初始的 Zygote 进程，保有所有的系统 Class 和 App 启动可能需要的资源，这样一来，就不需要启动一个 App 时，动态去加载相应的资源。

监听 /dev/socket/zygote socket.

```java
/**
 * Registers a server socket for zygote command connections
 *
 * @throws RuntimeException when open fails
 */
private static void registerZygoteSocket(String socketName) {
    if (sServerSocket == null) {
        int fileDesc;
        final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
        try {
            String env = System.getenv(fullSocketName);
            fileDesc = Integer.parseInt(env);
        } catch (RuntimeException ex) {
            throw new RuntimeException(fullSocketName + " unset or invalid", ex);
        }

        try {
            sServerSocket = new LocalServerSocket(
                    createFileDescriptor(fileDesc));
        } catch (IOException ex) {
            throw new RuntimeException(
                    "Error binding to local socket '" + fileDesc + "'", ex);
        }
    }
}
```

为何在 Android 中 fork Zygote 进程如此高效了？Linux Kernel 采用了 `Copy-On-Write` 的技术，`Copy-On-Write` 的意思是只有在写的时候才单独复制一份，而读的时候不进行操作。 换而言之 fork zygote 实际上并未实际复制什么东西，只有在发生写操作时，才单独复制一份。而另一方面，class 和 Resource 资源文件并不需要重新写，这些文件在绝大多数时候都是只读的。最后，实际的效果就是尽管运行着多个 APP，但实际只有一份 class 和 resource 文件在 Zygote 进程中。

#### System Server 和 系统服务

绝大多数 App 都是通过 fork Zygote 进程来完成了，只有一个例外，那就是 System Sever。在启动 System Server 之后，其他系统服务将自己注入到 System Server 中，同时其他系统服务也完成启动。这些服务主要包括，`Power Manager`,`Telephony Registry`,`Battery Service`,`Window Manager`等等。在这其中也包括我们熟知的 `Activity Manager`，`Activity Manager`在启动完成后，会发送一个 Intent.CATEGORY_HOME 的 intent，从而我们就能看到我们熟知的桌面。

![桌面样式](http://cdn.trendblog.net/wp-content/uploads/2014/12/android-launchers-2015.jpg)

由于之前对 System Server 进行过详细的讲解，这里就不在赘述，感兴趣的同学可以查看这篇文章 [Android Binder 全解析(2) -- 设计详解](http://www.woaitqs.cc/android/2016/05/26/android-binder-token.html)

#### 总结

上述章节把 Android 启动中发生的事情表述完毕，现在没有什么比下面这张图更能总结观点了。
![Android 启动](http://o8p68x17d.bkt.clouddn.com/android-boot.png)

----------------------

### 参考文献
- [Android Booting](http://elinux.org/Android_Booting)
- [Android booting sequence Explained](http://androidsrc.net/android-booting-sequence-explained/)
- [Understanding Android Boot Process](http://en.miui.com/thread-15659-1-1.html)

--------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期： 2016年6月16日
* 社交媒体： [weibo](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)
