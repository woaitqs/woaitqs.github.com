---
title: 编程随想
date: 2017-12-15 18:01:13
layout: post
description: "技术人该如何修炼自身的内功了？"
category: "tech"
tags: [tech, framework]
---

有段时间没有更新博客，以前都在工作业余时间更新，现在需求较多，时间就不富裕了，但主要原因还是懒吧，逃：）。

今天谈一些形而上学的问题，关于编程本身的一点看法，以及如何用这项知识，为自己谋得一些长远的利益。

<!--break-->

### 编程是给未来人讲故事

业务忙起来的时候，就容易把目光汇聚到需求上面，想着怎么能快速高效地解决问题。但我们所撰写的代码，却会长期存在，就像我们留下的书法作品，是会被后世点评的。当我们意识到这一点后，就会注意代码质量，架构设计，注释讲解，虽然短时间内会 delay 我们开发的进度，但对我们修炼内功还是很有好处。

眼界，很大程度上决定了你能前进的距离，就像「贫穷，限制了我的想象力」。

我从两个例子来说明这个问题。

#### 代码逻辑

有这么一个需求场景，如果应用没有获取到定位权限，需要在每个新版本升级的时候，弹出让用户去申请定位权限的对话框。

期初，有朋友这么实现了需求，伪代码如下代码。初看代码，好像没有什么太大问题，但实际上代码问题很多。如果我们从未来人的角度来看这段代码，会有什么样的想法呢？

* 代码没有任何注释，需要我们细细去体会着其中微妙的平衡。
* Location、Launch、Version 三者绑定在一起，就使得三者中如果任一出现问题，就会导致 BUG。
* 有单独的变量来记录是否展示了 GPS 对话框，但实际上这个变量，完全没有必要。
* isNewVersion 是不是每次版本升级都会变化了？如果用户直接升两个版本，会有问题吗？

```java
public static void tryToShowGpsDialog() {
    if (isLocationEnable() || !isShownGPS() || !isFirstLaunch() ||!isNewVersion()) {
      showGpsPermission();
      setShownGPS(false);
    }
}
```

这是修改后的代码，可读性和鲁棒性上面都有提高，先判断先验条件，最后更新条件与版本号放置在一起，逻辑清楚。

```java
public static void tryToShowGpsDialog(Context context) {

    // if location is able, skip it.
    if (isLocationEnabled()) {
        return;
    }

    // if launched, skip it.
    if (!isFirstLaunch()) {
        return;
    }

    // check version
    int latestShownVersion = getLatestShownVersion();
    int currentVersion = SystemUtil.getVersionCode(context);
    if (currentVersion > latestShownVersion) {
        // show dialog
        new GpsDialog(context).dialogShow();
        // update latest version
		  updateLatestShownVersion();
    }
}
```

#### 隐秘规范

代码世界里面，有许多不成文的规定，我们遵循这些规定，就能让别人更容易懂我们的代码。

* 目录结构是否清晰，读者是否很一眼就知道自己要去哪个地方找想要的东西了？「这里建议，用功能模块来进行划分」。
* 依赖反转。如果我们要做一个音乐播放器，别让代码里面充斥着各种状态判断，自己定义规则，通知外界来监听相应的状态变化。
* 理解清楚变化的与不变的，分离它们。变化的部分，多用接口，让变化的部分更容易扩展。

### 新技术正确对待

别低估新技术，后续的各种趋势，都是在新技术中诞生的。科技是第一生产力，对于出现的新词汇，不妨抱着谦虚的心，去了解下。如果碰到感兴趣的，也深入去学习，第一个吃螃蟹，你赚的可不只是新鲜感。

别太着迷新技术，技术基础决定上层建筑。花太多时间去尝鲜，也是浪费时间的做法，不如让自己的根基更扎实一点，更有可能性一些。

![螃蟹](https://static.pexels.com/photos/76966/crab-red-klippenkrabbe-grapsus-grapsus-shellfish-76966.jpeg)

有所追求，有所理智。

### 别低估领域知识的重要性

我们再开发一个需求的时候，容易流于表面。例如我们做一个下载的功能，可能就使用第三方库完事，但下载的细节很多，比如 `If-Range`, 如何快速中断下载线程等等。这是我们开发中很常见的场景。

我们往往会接触很多方面的知识，涉猎很多，但这样存在很大的问题。古人云，「擅百技，而专一长」，重点在于「专一长」。了解很多方面的知识，但都不深入的话，这样很容易被人替代，当有更年轻更低廉的人出现时，就岌岌可危了。Google、MicroSoft 等等大公司，都在建立自己的技术壁垒，我们技术人，也要建造自己的技术壁垒。技术人的价值，不在于有多广，而是在某些领域有多深。现在市面上，对音视频人才稀缺量很多，因为这个门槛没有一定的经验积累是达不到的。

深一点，再深一点。

![更深的地方更美](http://o8p68x17d.bkt.clouddn.com/diving.jpeg)

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2017年12月15日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------