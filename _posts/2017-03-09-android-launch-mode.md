---
layout: post
title: "Android Launch Mode 详解"
description: "Android Launch mode 详细说明"
category: "android"
keywords: "android, launch, mode"
tags: [android, launch, mode]
---
{% include JB/setup %}

我们所最常用的组件就是 Activity，Android 系统为了满足各种开发的需要，赋予了 Activity 四种不同的启动模式。虽然大家很可能都从各个地方了解过 Activity 的启动模式，但还是建议大家看看这篇文章，我会尽可能地理清思路，便于大家更好地理解启动模式。

<!--break-->

### 任务栈

对于 Android 而言，只能有一个 Activity 处于 Active 和 Visible 的状态，但我们通常由多个 Activity 的需求，那么如何来调度这些 Activity 呢。根据我们日常的使用习惯，使用栈来保存状态最合适不过了，完美符合了 LIFO 的要求，这个栈就是我们所说的任务栈。一般情况下，每个应用都会有一个默认名为包名的 Activity 任务栈。每个 Activity 都可以通过 Manifest 里面中的 [android:taskAffinity](https://developer.android.com/guide/topics/manifest/activity-element.html#aff) 来指定其所依附的任务栈。任务栈的名字取决于栈中最底部的那个 Activity 的 taskAffinity，如果没有指定，那就是包名。值得注意的是，当我们启动一个自定义 taskAffinity 的 Activity 时，并不会立即启动另一个任务栈哦，必须得加上 `NEW_TASK` 的这个 Flag 才行。

![android task](http://o8p68x17d.bkt.clouddn.com/task.jpg)

我们在日常的使用情况下，通常是通过桌面跳转到对应的 Activity 的，那么这种情况下，栈是怎么分布的了？是有一个全局的任务栈吗？非也！通过桌面跳转是进入了另一个任务栈，我们所熟知的导航区域（返回、HOME、任务列表）都用于辅助我们切换任务栈。我们看看任务列表的情况，就是下图，它所展示的不是一个一个进程，二是一个一个的任务栈。

![android task list](http://o8p68x17d.bkt.clouddn.com/task_manager.jpg)

1. Android 任务栈又称为 Task，它是一个栈结构，具有后进先出的特性，用于存放我们的 Activity 组件。

2. 我们每次打开一个新的 Activity 或者退出当前 Activity 都会在一个称为任务栈的结构中添加或者减少一个 Activity 组件，因此一个任务栈包含了一个 Activity 的集合, Android 系统可以通过 Task 有序地管理每个 Activity，并决定哪个 Activity 与用户进行交互，只有在任务栈栈顶的 Activity 才可以跟用户进行交互。

3. 在我们退出应用程序时，必须把所有的任务栈中所有的 Activity 清除出栈时,任务栈才会被销毁。当然任务栈也可以移动到后台, 并且保留了每一个 Activity 的状态. 可以有序的给用户列出它们的任务, 同时也不会丢失 Activity的状态信息。

4. 需要注意的是，一个 App 中可能不止一个任务栈，某些特殊情况下，单独一个 Actvity 可以独享一个任务栈。还有一点就是一个 Task 中的 Actvity 可以来自不同的 App，同一个 App 的 Activity 也可能不在一个 Task 中。

### Standard

接下来我们看看四种基本的启动模式，首先是标准模式，为了方便记忆，我把这种模式称之为「来者不拒」模式。只要有需求，就会新建一个 Activity 的示例，真是来者不拒呀。

1. Activity 可以有多个实例
2. 示例可以处于不同的任务栈中
3. 一个任务栈可以有多个实例

### SingleTop

这种模式也很简单，在启动 Activity 之前，会去查找对应的任务栈，如果在栈顶，就沿用这个 Activity，否则就新建一个实例。SingleTop 的适用场景很多，例如我们的消息列表页面，此时通知栏又收到一条消息的时候，点击通知栏的时候，我们是不希望新建一个 Activity 的实例的。

需要注意的是当 Activity 在栈顶的时候，触发的生命周期函数是 onNewIntent，且需要更新 intent，否则通过 getIntent 得到的还是原来的 intent。

![onNewIntent](http://o8p68x17d.bkt.clouddn.com/singletop.jpg)


### SingleTask

相对复杂的启动模式，当系统中（注意是整个 Android 系统中）存在 Activity 的实例时，会将该 Activity 之上的 Activity 纷纷出栈，并调用目标 Activity 的 onNewIntent 方法，否则新建一个实例。这里需要注意的是，根据文档描述，The system creates a new task and instantiates the activity at the root of the new task，但事实上并不是如此，只有你指定了 taskAffinity 的时候，才会给你创建一个新的任务栈，并将该 Activity 作为 Root 节点。

这里有一个额外比较恶心的地方是，当我们调用 startActivityForResult 的时候，4.x版本.会立刻在上个 Activity 中 onActivityResult 中返回一个为 cancel 的 resultCode.(不管新的activity是否是在新的任务栈中启动)
5.x版本修复了这个问题，不管是否定义了taskAffinity,都会把将要被启动的activity的启动模式忽略, onActivityResult方法会正常回调。

### SingleInstance

与 SingleTask 相似，但不同的地方在于 SingleInstance 要求在任务栈中有且只有它一个 Activity。

换句话说，A应用需要启动的MainActivity 是singleInstance模式，当A启动后，系统会为它创建一个新的任务栈，然后A单独在这个新的任务栈中，如果此时B应用也要激活MainActivity，由于栈内复用的特性，则不会重新创建，而是两个应用共享一个Activity的实例。

我们来想一个复杂的例子，在一个应用类，有三个 Activity，分别是 A，B，C，其中 B 是 SingleInstance 的，而 A 和 C 是标准的。A 启动 B 后，B 再启动 C，此时点击返回，会回到哪个界面呢？答案是 A，因为 B 是独立的栈，而 C 在分配的时候，会进入到默认的栈中去，此时默认的栈中是 A 与 C，点击返回的话，就将 C 弹出，此时就显示 A 啦。

--------------

### 文档信息

* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2017年3月9日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
