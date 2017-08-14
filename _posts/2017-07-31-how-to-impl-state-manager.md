---
layout: post
title: "利用状态机处理复杂业务逻辑(三)"
description: "遇到超复杂的业务逻辑，如何使得逻辑清晰呢"
category: "android"
tags: [android, ui, state]
keywords: "android, ui, state"
---
{% include JB/setup %}

在思考之下，这次决定写关于如何根据实际的代码情况来实现 Android 有限状态机。在这篇文章中介绍自己的一点理解，希望能对诸君有所帮助。

这片文章是从需求出发，自己所开发的状态机，需要支持什么功能呢？博主最近在做播放器相关的功能，对于有哪些需要，算是心知肚明。

<!--break-->

### 多维度的状态机

有限状态机不能完全满足播放器的需求，举个例子，有限状态机在任何时候只能有一个当前状态，显然这个设定对与播放器而言是不支持的。比如，播放器可以同时处于锁屏、播放状态，也可以同时处于暂停、锁屏状态。我们需要有限状态机能够拥有多个维度，在每个维度里面有唯一的活动状态即可。比如播放器，我们可以定义，`锁屏`,`播放`,`大小屏`多个维度，每个维度中定义不同的状态。在`大小屏`维度中，可以有`全屏`,`竖屏`,`小窗`三种状态，而在`播放`这个维度里面，又可以有`挂起`,`暂停`,`等待`多种状态。这里应该表述清楚了，简言之，Android 的自动机应该支持维度的概念，在每个维度里面，有唯一的状态即可。

### 如何实现这个状态机?

这个状态机，该如何为外界所用呢？这就需要定义较为通用的接口，而这是最难的点之一。为了达成这个目标，我们试着进行角色划分。思考一番，可以大概划分为三类角色。

- 状态
- 谁来切换状态
- 谁来响应状态变化

#### 一. 状态怎么定义

我们一个一个来看，首先是 `状态`, 在第一点中，我们提及到这个`状态`应该是有维度这个概念的，因为可以定义这样一个接口。

```java
public interface DimensionState {
    // 当前处于那个维度里面.
    int dimension();

    // 当前维度里面的状态.
    int state();
}
```

#### 二. 谁来切换状态

那么 `谁来切换状态` ？这应该算作一个比较复杂的业务逻辑才对，比如播放器内部状态切换、股票交易内部状态，这些都是和具体的业务相关的。我们只需要定义这样一种工具即可，内部逻辑通过这样的工具，就能实现状态切换了。Android 官方的 StateMachine 就是这样的一个工具。而现在我们需要针对我们的需求，去定义这样的一种工具。

由于我们这个状态机是有维度这个概念存在的，那么就先来定义每个维度里面是怎么处理状态的。

```java
public interface StateHandler {
    // 标明自己是属于哪一个维度的，初始状态是怎样的.
    void assgnToDimension(int dimension, int initState);

    // 提供的工具方法，可以实现两个状态之间的转换.
    void switchState(int preState, int currentState);
}
```

现在需要一个管理类，来控制着各个维度的 `StateHandler`，而对于用户而言，这个工具，还需要提供往外输出的接口。调用者可以注册相关的回调，以便监听相应的变化。

```java
public interface IStateManager {

    /**
     * 获取当前 Dimension 的 state.
     * @param clazz dimension.
     * @return 当前 state.
     */
    IState getState(Class<? extends IState> clazz);

    /**
     * 添加每个 dimension 相关的 handler.
     * @param stateHandler 每个域相关的处理器.
     */
    void addHandler(StateHandler stateHandler);

    /**
     * 当前消息，该被哪些 dimension 拦截。
     *
     * @param messageId message id.
     * @param info some kinds of dimensions.
     */
    void addInterceptInfo(int messageId, Class<? extends IState>... info);

    /**
     * 监听某个 dimension 的变化。
     *
     * @param clazz dimension.
     * @param l listener.
     */
    void registerListener(Class<? extends IState> clazz, StateChangedListener l);

    /**
     * 取消监听某个 dimension 的变化。
     *
     * @param clazz dimension.
     * @param l listener.
     */
    void unRegisterListener(Class<? extends IState> clazz, StateChangedListener l);

    /**
     * 通知某个 dimension 的当前变化.
     *
     * @param clazz dimension.
     * @param preState 以前的 state.
     * @param curState 当前的 state.
     */
    void notifyStateChanged(Class<? extends IState> clazz, IState preState, IState curState);

    /**
     * 重置状态机中里面的播放状态
     */
    void reset();
}
```

#### 谁来响应状态变化

现在是谁来响应这个状态变化了？我们定义一个 Processor，来处理状态的变化。

```java
public interface Processor {

    void preStateChanged(DimensionState preState, DimensionState currentState);

    void onStateChanged(DimensionState preState, DimensionState currentState);

    void postStateChanged(DimensionState preState, DimensionState currentState);

}
```

这个 Processor 会监听 StateManager 的变化，然后将这些变化通知出去，每个人去关注相应变化就好啦！

### 实际例子！！！

![农场经营](http://o8p68x17d.bkt.clouddn.com/former.jpeg)

我也认为前面的知识太过枯燥，就用实际的例子来进行说明。「农场经营」这个是个不错的例子，我们就一步一步进行展开。

- 状态定义

按照前面的说法，这里存在维度的概念。作zo我们分别定义 四季、耕作两个维度。

Spring Summer Autumn Winter

播种 施肥 收获

- 谁来切换状态

那么谁来切换状态呢？定义两个 Handler，分别是 SeasonHandler, ActionHandler。

SeasonHandler，会随着时间的变化，在四季中进行切换.

ActionHandler, 会和农民的操作相关，农民根据不同的作物，进行播种，然后施肥，最后收获。

- 谁来响应状态变化

这里可以有许多对象都可以来响应状态的变化，举一些例子哈。

农场经理，会在`春天`,`播种`开始前去购买种子，在`春天`和`夏天`的`施肥`阶段去购买化肥。

观光客，不管季节如何，会在`收获`后来农场吃瓜。

- 总结说明

通过这样的例子进行分析，我们可以不用写各种 if-else if - else if - else 类似的代码了。我们只需要针对自己所处于的角色，关注自己关心的维度，去关注这个维度的状态变化即可。这样的思想可以帮助我们简化逻辑，理清思路。


-----------------

### 文档信息

* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2017年8月14日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

-----------------