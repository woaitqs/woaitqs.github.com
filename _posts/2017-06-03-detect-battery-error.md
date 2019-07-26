---
layout: post
title: "应用耗电怎么办？"
description: "应用耗电怎么办？怎么定位问题了？"
category: "android"
tags: [android, battery]
keywords: "android, battery, profile, dumpsys"
---


厂商在预装你的 APP 之前，通常会进行功耗测试。这种功耗测试，与研发进行的测试不同，可不是通过 `ADB` 或者 `Android Studio Monitor` 进行验证的，而是通过特定的 `功耗仪` 来进行的。以前，厂商在进行测试的时候，会扣取手机的电池，并换上对应的仪器，通过这个仪器上的电流消耗，可以较准确地判断该应用是否有功耗问题。

那么，如果我们的应用被厂商告知，有性能问题时，又不提供相应的仪器(`测量方式一般为机密`)情况下，如何快速定位功耗问题了？

<!--more-->

---------------

### 定性分析

Android 开发就免不了与[系统服务](http://www.woaitqs.cc/android/2016/06/21/activity-service)打交道，功耗相关的服务也是属于其中的一种。

> adb shell dumpsys [system service name]

用法如上所示，非常简单，举几个例子。adb shell dumpsys activity 显示 ActivityManager 相关的信息，adb shell dumpsys alarm 显示的是 Alarm 服务相关的信息。power 则是系统的电源管理服务，我们能否通过 `dumpsys power` 获取到需要的信息吗？

不过很遗憾，这里面提供的信息并不是很足够。这个 power 列举了电源相关的很多东西，大多都用不上。不过有个地方，还是能帮助到我们的。

WakeLock Summary。wakelock 是告知系统还有服务需要进行，只有有一个 wakelock 存在，系统就不能进入睡眠中。通过这个可以判断是否有程序调用了这个，而并没有释放，这可能是导致功耗问题的原因之一。

在后续的调研中，发现了另一个很好用的系统服务`batterystats`，这个服务提供的信息远多于`power`, 而用法也并不难。

1. adb shell dumpsys batterystats --enable full-wake-history；容许开启完整的 wake 历史。

2. adb shell dumpsys batterystats --reset；清空以前的信息

3. 在应用上进行一段时间的操作

4. adb shell dumpsys batterystats [your package name]> sample.txt；这样所有的信息就在这个 sample 文件中了。

是不是很棒啊？23333

我们来分析下，得到的结果。结果主要分为 `Daily stats`,
`Statistics since last charge`, `Device battery use since last full charge` 以及 `UID info`。这里只用关心，UID 相关的信息即可。

```shell
u0a151:
  Wake lock *alarm* realtime
  Wake lock AudioMix realtime
  Wake lock *launch* realtime
  Wake lock WindowManager realtime
  Sensor GPS: (not used)
```

在这个信息块中，将记录应用进行了哪些消耗电量的操作。在上面的例子中，可以看到 AlarmService、AudioMix 等都消耗了电量，而没有使用传感器。

对这个管兴趣的同学，可以继续看这个链接。[https://commonsware.com/Android/previews/power-measurement-options](https://commonsware.com/Android/previews/power-measurement-options)

---------------

### 定量解决

前面提供了一些定量分析的方法，但这些信息还是太泛，我们要的是具体的解决方案，需要从代码上解决功耗问题。那么我们就细细地思考下，如何来分拆这些问题。

1. CPU 消耗过多

2. 网络请求

3. 传感器与屏幕唤醒

耗电通常是由上诉三个方面引起的，我们一个一个地突破它们。

首先是 `传感器与屏幕唤醒`, 这个可以通过前面提到的方法，找到应用是否在该时间段内调用过传感器，或者 wakelock。如果确实存在这样的问题，就直接在项目中查找，关于 GPS、wakelock、Sensor 等等的调用，在找到源头之后，进行合适的处理即可。

然后是 `网络请求`, 我们得借助于 Android Monitor，找到其中的 `Network` 区域，在这区域中，如果发现确实还有网络波动，说明在此时进行了不应该发生的网络请求，从而导致在厂商这里没有通过测试。如果项目，都是使用统一的网络模块，那就在源头处加上日志、或者断点调试等方式，来帮助我们定位问题。

![monitor network](http://o8p68x17d.bkt.clouddn.com/monitor_network.png)

最后才是 `CPU 消耗过多`, 这次厂商提及的问题就是放生在这一块。告知我们在保持屏幕不动、未进行操作时，仍有电量消耗。通过 Monitor 中的 `CPU` 区域，确实发现有相应的活动。此时，一个伟大的神器就帮助了我，那就是 `TraceView`。这个工具实在太好用，关于这个工具的用法，也有很多教程，就不再赘述了。

总之，是想告诉大家分析问题的方法，遇事不慌，冷静下来，总有解决的方法。

--------------

### 文档信息

* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2017年6月3日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

--------------
