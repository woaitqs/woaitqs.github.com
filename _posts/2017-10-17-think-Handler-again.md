---
title: 重读 Handler
date: 2017-10-17 19:54:10
layout: post
description: "重读 Handler"
category: "android"
tags: [android, handler, ui]
---

# 重读 Handler
有太多东西经历时间的考验后，还依然释放着光彩。这些光亮的东西，将在我们迷茫的时候指引我们方向。

<!--more-->

### Handler TM 到底是干啥的

选择再单独写一篇关于 Handler 的文章，不是为了炒旧饭，而是这个机制实在重要，以及对这个机制的更深入理解，能帮助我们点开别的技能树。每当你过一段时间，站在不同的立场上去看以前的一些东西，就能更加透彻地理解那些珠玑。

我所理解的 Android 开发，分为三大要素：「系统服务」、「组件」和「通信」。日常进行的开发工作，就是通过「组件(Activity、View等等)」与「系统服务」进行「通信」，从而达成预期的效果。而「通信」分为进程间通信(Binder) 与线程间通信(Handler)，这篇文章讲的还是线程间通信(Handler)。

如果我们来设计这么一个线程间通信的机制，我们来如何实现了？线程间通信需要保证两方面的问题，效率和安全。首先是效率上面，由于线程间访问共享内存的速度都比较快，只要没有特别耗时的操作，效率方面都是可行的。另一方面是安全，线程间通信时需要处理「同步」的问题，保证数据读取的正确。

在设计的时候，要使得线程间通信尽量地让交互的线程分离，这也是非常常见的职责分离。如果一个线程需要和另一个线程交互，这就会使得代码紧耦合，日后维护非常麻烦。因此需要剥离出一个第三方的东东，它去负责处理「效率」与「安全」，线程只需要与这个东东进行交互即可。嗯嗯，这不就是我们的「Handler」吗？

<!--more-->

### 那么接口来如何定义了？

先假装世上没有「Handler」这种东西存在，你来怎么定义这个通信模块呢？既然通信嘛，那肯定是有发送和接收两部分滴。

发送接口：
	1. 发送实物
	2. 发送时间

> handler.sendMessageAtTime(Message msg, long uptimeMillis);

其他接口都可以通过这个方法来变形实现，举例如下：

```java
public final boolean post(Runnable r) {
  return  sendMessageDelayed(getPostMessage(r), 0);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis) {
  if (delayMillis < 0) {
     delayMillis = 0;
  }
  return sendMessageAtTime(msg,SystemClock.uptimeMillis() + delayMillis);
}
```

接收接口：
这个的定义就要相对复杂一些，难点在于这个是接收消息的函数是被动调用的，需要提前定义好。这种通过注入的方式实现的东西，一般都是接口。

```java
public interface Callback {
  public boolean handleMessage(Message msg);
}
```

这样，我们需要接收消息时，只要注入这个接口就可以了。

### Handler 的基本实现原理

在这里需要提前提出一个概念，生产者消费者模式。这个模式，在很多地方都被提及，就不再描述了。通常的生产者消费者模式，是多对多的，Handler 对这个进行了简化，变成一对一。一个线程生成消息，另一个线程消费消息。

![pipeline](https://i.loli.net/2019/03/04/5c7c89858b91b.jpg)

要想做好这件事情，需要考虑几方面的事情？
1. 消息是怎样的格式？怎样才能完全覆盖我们的需求？
2. 怎么保证消息的安全，不会发生丢失或者错误？另外又怎么保证安全了？
3. 怎样去获取消息呢？发送是主动的事情，接收可是被动的事情。

这三个问题的答案分别是 `Message.java`, `MessageQueue.java`, `Looper.java`。

#### 怎么抽象消息的 ？

首先看看 Message 的定义。Message 即为可传递的消息，要想做到足够通用，就得拥有足够的表达能力。那么怎么达到这个目的呢？这就考量我们抽象的能力了。

自身数据与外部数据。自身数据的格式比较简单，只要能够唯一标示自身就好了。外部数据从两方面来分类，一是数据，二是行为。数据使用 object 表示，行为由 runnable 表示，这样的表达性就 很高了。

实际上 Message 的实现，也和我们设想的一样。

```java
// 外部数据 - 数据
/*package*/ Bundle data;
// 所属于哪一个 Handler
/*package*/ Handler target;
// 外部数据 - 行为
/*package*/ Runnable callback;
```

置于 what、arg1、arg2、obj 等等都只是一层糖衣，方便我们进行操作而已。

#### 怎么保证消息的安全和正确？

这个功能主要是由 MessageQueue 来实现的，涉及到较多的 native 细节，这里就不展开说。实际上 MessageQueue 可以有很多种实现方式，其中 Java 层较为出名的就是 BlockingQueue。

我们从生产者消费者的概念上来看这个问题，生成者制造的东西过多时，填满了整个队列，此时就不能进行生产，不然东西放在哪了？同理，如果队列为空，那么消费者就不能进行消费了，总不能消费空气吧。

从上面直白的描述上，可以看到这个消息队列主要保证了什么。当队列为空，或者队列满的时候，都会阻塞，这样就能强制地保证正确和安全性。

其对应的接口，也较为简单。

```java
// 消息入队
boolean enqueueMessage(Message msg, long when);

// 轮询消息
Message next();

// 移除消息
void removeMessages(Handler h, int what, Object object)
```

#### 如何接收消息的？

既然消息的接收是被动的，那么就只能通过轮询的方式，来查看是否有新的消息。Looper 就是因此设计出来的轮询器。

Looper 在开启后，将进入一个死循环中，不断地通过 `queue.next()`进行查询，如果有这个消息，那么根据 Message 中的 target ，来执行 `dispatchMessage`，否则继续进入循环中进行查询。

#### Handler 扮演着什么样的角色了？

 前面提及了 `Message.java`  `MessageQueue.java`  `Looper.java` 三者，Handler 就是负责他们之间的统筹协调。

Handler 在启动时，会检查当前线程是否有 Looper 在工作，并在构造函数中定义怎么处理消息。随后，将 Handler 作为 Message 一部分，交由另一个线程进行消息发送。

-----------------

### 文档信息

* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2017年10月17日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

-----------------
