---
layout: post
title: "利用状态机处理复杂业务逻辑(二)"
description: "遇到超复杂的业务逻辑，如何使得逻辑清晰呢"
category: "android"
tags: [android, ui, state]
keywords: "android, ui, state"
---


在[上一篇文章](http://www.woaitqs.cc/android/2017/06/20/using-state-manager-to-deal-with-complete-ui-features)中，介绍了在开发中，多状态情况下，View 需要处理的情形较多，一般的书写方式逻辑复杂，很容易出问题。其后，引入了自动机的概念，在这篇文章中，将对有限自动机进行更为详细地说明。

<!--more-->

### 火箭发射的例子

无意中，看到一篇文章，[使用状态机的好处](https://github.com/nixzhu/dev-blog/blob/master/2015-04-23-state-machine.md)，这篇文章中引入了一个更为简单的例子，火箭发射。在这里我就也使用这个例子，进行说明。

![state_machine_rocket](http://o8p68x17d.bkt.clouddn.com/state_machine_rocket.png)

如图，这就是我们需要实现的火箭，火箭上面有两个按钮，分别是`点火`，`中断`。`点火`按钮只有在火箭初始状态，或者重置状态后才能实现，其他状态（例如倒计时状态）都不能点击。`中断`只有在火箭已经进入准备状态，并且未起飞的情况下才能使用。

这是我们日常开发中的一个缩影，在其他地方也很常能见到类似的例子。例如`发布`按钮只能在文案不为空，且字数在特定范围内，且未向服务端发送请求时，才能被使用。通常情况下，我们用 `if-else-if` 类似的代码来解决这样的问题了。那么能否有更好的方式呢。

### 有限自动机处理火箭发射

我们可以将上述例子分拆为两部分、一部分是状态变化，一部分是状态响应。火箭自身的状态会发生变化，在状态发生变化后，响应的状态响应也会发生变化。

有限状态机的切换，一般可以由状态装换图来进行表达。左侧是条件，顶部是状态。其他的每一项是，相应状态在该条件下发生的变化，置灰则表示该情况不容许。这么来看，是不是很清晰了？

![rocket_table_state](http://o8p68x17d.bkt.clouddn.com/rocket_state.png)

事情就变得相对简单了，用状态机去切换状态，响应的按钮响应对应的状态变化即可，设想中的代码如下：

```java
private void initStartButn() {
    startBtn
        .bind(Status.IDLE, enable)
        .bind(~Status.IDLE, disable);
}
```

### 扩展思路

上述例子，是最简单的例子，我们遇到的实际情况可能会更复杂。因而，我们需要对这些状态进行进一步的整理和分类，这里就有两种思路。

1. 分层。如果大家对 Android 的[事件处理](http://www.woaitqs.cc/android/2016/03/05/android-touch-system)比较熟悉的话，一定了解其中的分层概念。对于状态处理，同样也可以执行类似的操作，可以针对状态进行分发、拦截和消费。

2. 分类。状态也是可以有分类的，例如对于一个播放器而言，有播放状态(暂停、继续等等)、播控状态(锁屏、loading等等)，对于一个特定的控件而言，应该只关心一个或者几个分类下的状态，这样更有利于我们进行状态管理。

在下篇文章中，可能会在 Android状态机的实现，和自定义状态机实现 中选择一个进行说明，下期见。

p.s. 项目压力大，加之一堆私事，最近更新少，各位老板见谅。

-----------------

### 文档信息

* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2017年7月10日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

-----------------
