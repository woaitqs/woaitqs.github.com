---
layout: post
title: "Android Gradle 常用命令 -- Gradle教程(四)"
description: "Gradle 系列教程"
category: "gradle"
keywords: "gradle, android, 教程"
tags: [android, gradle, build]
---
{% include JB/setup %}

在了解 Gradle 的基本情况和 Groovy 的基本用法后，现在来看看有哪些常见的 “Android 轮子” 可以直接使用的。在 Gradle 的官方页面中并没有查找到与 Android 相关的详细文档，反倒是 Android 官网找到了，这里把地址列出来，有兴趣的同学，可自行前往学习[https://developer.android.google.cn/studio/build/index.html](https://developer.android.google.cn/studio/build/index.html)。

<!--break-->

------------------------

### 依赖配置

在开放 App 的时候，常会依赖到第三方库、或者公司内部开发的 SDK。这时候就要通过 Gradle 提供的依赖管理来帮助我们。依赖管理主要分为两种情况，一是本地依赖、二是远程依赖，让我们分别看看两种情况。

1. 依赖某个Module

```groovy
compile project(':lib:rpc')
```

2. 依赖本地某些文件

```groovy
compile files('lib/md5.jar')
compile fileTree(dir: 'libs', include: '*.jar')
```

对于远程的依赖，需要借助一些标示，来帮助我们唯一地确认某个远程依赖。Gradle 采用的是这样一种命名方式，`group + name + version + ext(optional)`。group 是指所属的机构，在 Android 开发中 group 也可以等同于包名，name 是指对应的名字，version 则是版本，ext 是文件的格式，可省略。

了解了标示是如何组成之后，那么还有一个问题，从哪里去或者这些远程内容呢？我们将远程存放这些内容的地方称为仓库，我们可以同时声明多个仓库。

```groovy
repositories {
  mavenCentral()
  maven { url "https://jitpack.io" }
}
```

3. 远程依赖

```groovy
compile('org.apache.ant:ant-launcher:1.9.6@jar')
compile('org.codehaus.groovy:groovy:2.4.7')
```

------------------------

### 签名

相信你已经对 Android 如何进行签名有所了解了，这里就不介绍了，直接看看 Gradle 提供的打包方法。

```groovy
android {
    ...
    signingConfigs {
        release {
            storeFile file("my-release-key.jks")
            storePassword "password"
            keyAlias "my-alias"
            keyPassword "password"
        }
    }
}
```

这里对上面的代码做一些简单的说明，在 signingConfigs 接受闭包，这里 release 是指打包的方式，同样也可以针对 debug 做另外的签名处理。

签名主要是这样组成的。

1. 一个Keystory
2. 一个keystory密码
3. 一个key的别名
4. 一个key的密码
5. 存储类型

分别和上面的签名配置相同。

------------------------

### 文档信息

* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2017年1月3日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
