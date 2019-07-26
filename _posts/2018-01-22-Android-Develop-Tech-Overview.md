---
title: Android Develop Tech Overview
date: 2018-01-22 13:19:28
layout: post
description: "Android 开发过程中可能涉及到的知识，大体进行系统的归类，便于日后查漏补缺，不断进步"
category: "android"
tags: [tech, framework, android, overview]
---

在前面写过一篇文章，[Android 系统学习计划](http://www.woaitqs.cc/2016/06/16/android-system-learning/)，在这篇文章中罗列了Android的几个方面，但很不周全。于是新起一个炉灶，来系统地归类下 Android 相关的领域知识，便于日后查漏补缺，不断进步。

但即便如此，还是不能涵盖所有方面，不足之处还望海涵。

<!--more-->

----------------------------------------

## 基础组件

个人对 Android 开发的总结是，通过`通信组件`和`系统组件`进行通信，得到相应的内容或者能力，并最终在`应用组件`上进行展示。地图应用通过 Binder 与系统的 LMS 进行交互，得到相应的地理位置，并最终在 MapView 上进行展示。又比如，TimeLine 软件通过 Binder 和系统网络服务交互，拿到通信能力并和服务端通信，得到对应的数据后，RecycleView 显示到界面上。

#### 系统组件

1. 各类系统服务

    a. ServerManager

    b. PackageManagerService

    c. ActivityManagerService

    d. WindowManagerService

    e. others...

2. 进程

    a. Zygote 进程

    b. 应用进程启动

> #### 相关笔记
> 1. [Android 应用进程启动流程](http://www.woaitqs.cc/2016/06/21/activity-service/)

#### 通信组件

1. Binder

    a. Binder 架构设计

    b. AIDL 领域语言

> #### 相关笔记
> 1. [Android Binder 全解析(1) -- 概述](http://www.woaitqs.cc/2016/05/23/android-binder/)
> 2. [Android Binder 全解析(2) -- 设计详解](http://www.woaitqs.cc/2016/05/26/android-binder-token/)
> 3. [Android Binder 全解析(3) -- AIDL原理剖析](http://www.woaitqs.cc/2016/05/30/android-binder-proxy-and-token/)

2. Handler

    a. Looper、Handler、MessageQueue 工作流程

> #### 相关笔记
> 1. [重读 Handler](http://www.woaitqs.cc/2017/10/17/%E9%87%8D%E8%AF%BB-Handler/)
> 2. [Android Handler机制全解析](http://www.woaitqs.cc/2016/06/06/android-handler/)

3. 数据封装格式

    a. Bundle

    b. Parcelable

    c. Intent

#### 应用组件

1. 四大组件

    a. Activity、Receiver、Service、BroadCast

    b. 各组件的生命周期

> #### 相关笔记
> 1. [Android Activity 生命周期是如何实现的](http://www.woaitqs.cc/2016/07/19/how-activity-lifecircle-work/)

2. 界面显示原理

    a. Vsync 机制

    b. Choreographer

    c. Window

3. 事件处理机制

> #### 相关笔记
> 1. [Android 事件处理机制分析](http://www.woaitqs.cc/2016/03/05/android-touch-system/)

4. 视图组件

    a. ViewGroup (LinearLayout, FrameLayout, etc...)

    b. View (TextView, ImageView, etc...)

    c. RecyclerView / ListView

    d. ViewGroup

    e. CoordinatorLayout

    f. Fragment

5. 自定义 View

    a. onMeasure、OnDraw、OnLayout

> #### 相关笔记
> 1. [Android View 全解析(一) -- 窗口管理系统](http://www.woaitqs.cc/2016/10/10/android-view-theory-1/)
> 2. [Android View 全解析(二) -- OnMeasure](http://www.woaitqs.cc/2016/10/18/android-view-theory-2/)
> 3. [Android View 全解析(三) -- onLayout](http://www.woaitqs.cc/2016/10/25/android-view-theory-3/)
> 4. [Android View 全解析(四) -- onDraw](http://www.woaitqs.cc/2016/10/28/android-view-theory-4/)
> 5. [Android View 全解析(五) -- 实践自定义 View](http://www.woaitqs.cc/2016/11/28/android-view-theory-5/)

6. 动画

    a. 属性动画

    b. 变换动画

    c. 矢量动画

#### 数据持久化

1. SharedPreference

2. DataBase

    a. ORM 框架

----------------------------------------

## 应用构建

#### Android APK 组成

#### 插件化

#### 代码保护

#### Gradle 打包工具

----------------------------------------

## 应用体验优化

#### 网络优化

1. 速度

2. 弱网

3. 安全 

> #### 相关笔记
> 1. [移动 APP 网络优化概述](http://blog.cnbang.net/tech/3531/)

#### 性能监控

1. 开发者选项

2. 卡顿监控

> #### 相关笔记
> 1. [广研Android卡顿监控系统](https://mp.weixin.qq.com/s/MthGj4AwFPL2JrZ0x1i4fw)
> 2. [Android 启动卡顿原因查询](http://www.woaitqs.cc/2016/11/08/run-your-app-fast/)

3. 电量监控

> #### 相关笔记
> 1. [应用耗电怎么办？](http://www.woaitqs.cc/2017/06/03/detect-battery-error/)

4. 流量监控

#### 性能优化

1. Java GC 的原理

2. 内存抖动

3. Memory Leak

4. 图片内存

5. 更合适的集合工具

#### 应用保活

----------------------------------------

## 架构

#### Reactive

#### Clean 架构

----------------------------------------

#### 开源

1. Volley

2. Glide

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2018年1月22日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------