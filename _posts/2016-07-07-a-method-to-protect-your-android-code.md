---
layout: post
title: "Android 代码保护攻防战，以及一种别样的技巧"
description: "Android 保护代码的一种进阶技巧，利用官方的 AssertManager 和 加壳的方式来做相应保护的事情"
category: "android"
keywords: "android, 混淆, 反编译"
tags: [android, 混淆, 反编译]
---
{% include JB/setup %}

<div style="border:solid 1.5px #ccc;padding:20px 20px 10px 20px;margin-bottom: 20px;border-radius: 6px;">

          <span style="position: relative;padding: 5px 15px 5px 15px;background-color: white;bottom: 30px;font-size: 16px;font-weight: bolder;">摘要</span>

          <span style="position: relative;display: block;bottom: 15px;">
          本文介绍了 Android 代码打包过程，以及基本的反编译技巧，在最后介绍了一种别样的方法用来保护你的代码，值得阅读。
          </span>
</div>

在某个风不平浪不静的日子里，接到了一个技术调研的任务，这个技术相对而言毕竟繁琐且门槛很高，于是我在网上搜寻相应的应用，看是否能从他们混淆后的代码中得到一些思考。在经历一段波折的反编译后，虽然对调研需要的内容没有起到太大的作用，反倒是学到了一些如何保护代码的不太常见的技巧，大有无心插柳柳成荫的意思，于是专门写这篇文章分享给大家。

<!--break-->

![无心插柳](http://image.datatang.com/news/2015/4/3/130725029301353521.png)

如果对应用打包过程和常见反编译方法都有所了解，可以跳过这两部分，直接看最后的章节.

------------------------

### Android 打包过程

要进行逆向工程，反编译项目，那么首先得知道顺向工程是如何进行的，即我们得知道怎么打包的。Android 的打包过程是通过各种工具将你的工程项目转换成 APK 格式的可安装程序，这个过程其实非常灵活，在每个步骤可以做很多自定义的事情，甚至你可以完全手动完成整个步骤，正是这样灵活的过程也给许多黑色产业可乘之机。接下来我们看看具体的打包过程。

![android build process](http://o8p68x17d.bkt.clouddn.com/android_build_process.png)

#### 1) 打包 res 目录下的资源

这里用到了 Android Asset Packaging Tool (aapt) 这个工具，如同名字所述，这个工具负责对 Android 中用的资源进行打包，实际上这个工具能提供的功能还不只打包，还可以查看 APK 相应的信息(权限、资源表、label和Icon 等等)，也可以对资源进行增删改查。

这里列举一些参见的用法，对 AAPT 感兴趣的，可以通过 `aapt --help` 的进行了解。

```shell
# 查看前面10个资源文件
$ aapt list weibo.apk | head -n 10

AndroidManifest.xml
assets/AZURE.png
assets/BLUE.png
assets/CYAN.png
assets/Common.zip
assets/GREEN.png
assets/LineRound.pvr
assets/MAGENTA.png
assets/MusicAssets.zip
assets/MusicVideoAssets.zip
```

```shell
# 查看 APK 权限
$ aapt dump permissions weibo.apk

package: com.sina.weibo
permission: com.sina.weibo.data.sp.dbsharedpreference
uses-permission: com.sina.weibo.data.sp.dbsharedpreference
uses-permission: android.permission.AUTHENTICATE_ACCOUNTS
uses-permission: android.permission.WRITE_SYNC_SETTINGS
uses-permission: android.permission.MANAGE_ACCOUNTS
uses-permission: android.permission.READ_SYNC_SETTINGS
uses-permission: android.permission.USE_CREDENTIALS
uses-permission: android.permission.READ_SYNC_STATS

...

uses-permission: android.permission.GET_ACCOUNTS
```

```shell
# 打包资源到对应的APK中去，并生成相应的 R 文件。

aapt p[ackage] -f -S <res路径> -I <android.jar路径> -A <assert路径> -M <AndroidManifest.xml路径> -F <输出的包目录+包名>

```

#### 2) AIDL 的编译

(Android Interface definition language)AIDL 是方便我们进行进程间通讯的语法糖。Android 同样提供了工具来方便我们进行对 AIDL 的编译，在 SDK 中的 build-tools 目录下。具体与 AIDL 相关的知识，可参看 [Android Binder 全解析(3) -- AIDL原理剖析](http://www.woaitqs.cc/android/2016/05/30/android-binder-proxy-and-token.html)

#### 3) 生成对应的 class 文件，将 Class 文件进行打包

我们书写的是 Java 文件与平台无关，但实际运行环境各有差异，于是需要生成相应的中间文件来帮我们屏蔽这些差异，对于 java 而言这就是 Class 文件。这一步完成的工作就是将 R 文件、AIDL 生成的代码和 src 中的java文件编译成 class 文件。JDK 中就带有这个工具，Java programming language compiler(javac)。

但 Android 采用的虚拟机不是 JVM，而是针对移动设备进行优化后的 Davlik 或者 ART 虚拟机，不能简单地使用 class 文件，而是需要将 class 文件转成 smali 格式文件，这个步骤是通过 `android-sdk/platforms/android-X/tools/lib/` 目录下的 dx  工具生成 `classes.dex` 文件，这个 Dex 文件就含有工程中的所有 java 文件。

题外是说一句，大家可以看看 Scala 等同样基于 JVM 来实现的语言，这样会更明显地意识到当初虚拟机这套设计带来的价值。

#### 4) 打包并签名

通过 apkbuilder 工具将所有 dex 文件、编译后的资源一并打包到 apk 文件中。但是此时生成的 APK 包不具备签名，是无法在手机上安装的。具体的签名流程，可以在 [Sign Your App](https://developer.android.com/studio/publish/app-signing.html) 里进行查看。

在开发过程中，主要用到的就是两种签名的 keystore 。一种是用于调试的 debug.keystore，它主要用于调试，在 Eclipse 或者 Android Studio 中直接 run 以后跑在手机上的就是使用的 debug.keystore。另一种就是用于发布正式版本的 keystore。

在签名结束后，使用 zipalign 进行相应 APK 的优化，至此这个打包过程结束。

最后这个 APK 目录所包含的内容就如下表所示：

![apk 组成](http://o8p68x17d.bkt.clouddn.com/apk_compoent.png)

------------------------

### 常见反编译方法

知道打包是怎么进行后，就可以办轻松地进行反编译的工作，我们感兴趣的内容无非是其中的资源和其中的代码，根据前面提到的打包知识，代码主要在 classes.dex 文件中，因此我们需要做的是两件事情，一是得到被压缩处理的资源文件，二是从 classes.dex 文件中得到相应的代码。

#### 1) 获取对应的资源文件

这里需要使用到 [Apktool](http://ibotpeaches.github.io/Apktool/) 这个工具来帮助我们完成对应的获取资源的任务。这个工具只要有两个功能，一是反编码资源文件，使得其尽量具有高的可读性；二是解析 dex 文件，得到可读性稍微高点的 smali 文件。smali 文件就类似于 JVM 中的 class 文件，这里的 smali 文件就是 Davlik 中的 class 文件，其具体的语法可以参考链接 [http://androidcracking.blogspot.hk/search/label/smali](http://androidcracking.blogspot.hk/search/label/smali)

apktool 有非常丰富的语法，支持较多的功能，这里只是简单地说一下反编译的用法。

```shell
# apktool d -o filePath apkPath
# 解析 APK 的资源到指定的 filePath 中去.

$ apktool d -o test.out test.apk
I: Using Apktool 2.1.1 on test.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: 1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```

反编译后，会得到如下所示的 APK 文件，在这种情况下，就可以查看 AndroidManifest 文件，以及 res 目录下的各类文件。

![apk files](http://o8p68x17d.bkt.clouddn.com/apk_files.png)

#### 2) 解析 classes.dex 文件

APK 包本事就是一个 zip 包，所以直接将后缀名改成 zip，就可以进行解压了，解压后就能看到里面的 classes.dex 文件，现在的工作就是讲 classes.dex 转成日常可读的 jar 文件。这里就需要用到另一个工具 [dex2jar](https://github.com/pxb1988/dex2jar), 这个工具在绝大多数情况下都能满足我们的需要。

```shell
./d2j-dex2jar.sh classes.dex
```

在这个步骤后，就能得到 `classes-dex2jar.jar`，为了更方便地阅读 jar 文件，可以使用 [JD-GUI](https://github.com/java-decompiler/jd-gui) 进行代码阅读。

![JD-GUI](https://camo.githubusercontent.com/8286f65f4b148a27de05a78fa366074543e89ce3/687474703a2f2f6a642e62656e6f772e63612f696d672f73637265656e73686f7431372e706e67)

------------------------

### 保护代码资源

现在知道了打包过程，和反编译的基本用法，那么如何保护自身呢？如何使得尽可能地将自己的内容保护好？没有绝对的安全，就像《美国队长3》里面说的, `有经验有耐心，是没什么做不到的`，我们只有尽可能地增加反编译的难度，才能尽可能地保护好自身。解决这些问题，最好的做法，可能是类似于 iOS 的做法，把这些过程完全纳入自身控制中。

#### 1) 资源的混淆

在使用资源的时候，都会尽量用一些可读性较高的命名，例如 `layout/card_list_item.xml`，使用 apktools 后就能看到这些名字，那么反编译的人就能按图索骥地找到对应的逻辑。对资源命名进行混淆后，能改善这种状况。

混淆主要是将有意义的名字变成无意义的代号，这样反编译者就不能从中得到什么有用的信息，例如

> R.string.name   -> R.string.a   
res/drawable/icon -> r/s/a

目前对资源进行混淆，主要方案是基于 resources.arsc 的修改。[微信Android资源混淆打包工具](http://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=208135658&idx=1&sn=ac9bd6b4927e9e82f9fa14e396183a8f#rd) 中详细地说明了原理，有兴趣的同学可以去看看。

笔者针对某个 APK 进行了资源混淆，不仅对资源进行了混淆，也使得包大小减少了不少，有兴趣的可以试试，效果如图所示。

![混淆后的资源文件夹](http://o8p68x17d.bkt.clouddn.com/apk_resources_folder.png)

![资源混淆前的包大小](http://o8p68x17d.bkt.clouddn.com/apk_size_before_confusion.png)

![资源混淆后的包大小](http://o8p68x17d.bkt.clouddn.com/apk_size_after_confusion.png)

#### 2) 代码的混淆

资源可以进行混淆，代码更是可以，而且代码混淆由来已久，也有完备的解决方案。目前主要采用了 Progruad 来进行，Proguard 通过删除无用代码，将代码中类名、方法名、属性名用晦涩难懂的名称重命名从而达到代码混淆、压缩和优化的功能。关于 Progruad 用法的文章，可以参考这个链接 [Android中ProGuard配置和总结](http://treesouth.github.io/2015/04/05/Android%E4%B8%ADProGuard%E6%B7%B7%E6%B7%86%E9%85%8D%E7%BD%AE%E5%92%8C%E6%80%BB%E7%BB%93/)。这里就不再详细地说明用法，网上的教程足够多，直接看看混淆后的效果。

![代码混淆](http://o8p68x17d.bkt.clouddn.com/code_confuse.png)

#### 3) 一种别样的技巧

尽管进行了混淆，但有些类由于特殊性是无法被混淆的，即便被混淆了，也能通过 java 中的一些方法调用看出蛛丝马迹。那么还有什么好的方法吗？可能有人说，通过插件的方式将代码从网上下载下来，这样也不是很好，比较你的核心逻辑也喜欢在无网的时候可以使用。于是，就想到一种方法，将代码在 APK 包里面藏起来，再动态加载。

这里先提及在 PC 时代，被广泛使用的一种技巧 -- 加壳，这种技巧常被用于病毒和软件保护。

> 利用特殊的算法，对可执行文件里的资源进行压缩，只不过这个压缩之后的文件，可以独立运行，解压过程完全隐蔽，都在内存中完成。它们附加在原程序上通过加载器载入内存后，先于原始程序执行，得到控制权，执行过程中对原始程序进行解密、还原，还原完成后再把控制权交还给原始程序，执行原来的代码部分。加上外壳后，原始程序代码在磁盘文件中一般是以加密后的形式存在的，只在执行时在内存中还原，这样就可以比较有效地防止破解者对程序文件的非法修改，同时也可以防止程序被静态反编译。

将代码藏起来的方式，也就是在 Android 上使用加壳技术。那么首要问题是在哪里加壳？Android 官方提供的 assert 目录，就是一个比较合适的地方，其后可以通过 AssertManager 来进行访问。另一个问题是对什么加壳，这里可以使用 jar 包，Android 是可以动态加 jar 包加载进入到 classLoader 中的。最后一个问题是怎么藏起来，我们可以将 jar 包藏在某个具有虚假的资源中，例如 icon.png 下，换而言之是，看上去是 png ，实际也可以打开，但在其中有被隐藏的 jar 文件。

有不少现成的工具，可以使用这样的技术，可以实现将图藏起来的功能。我在这里，找到 sf.net 上面的一个工具, [hide in picture](https://sourceforge.net/projects/hide-in-picture/)，可以藏任何文件，并且支持对这些文件进行加密，也就是说没有密钥，就算知道这样的加壳技术，也不见得能够得到被隐藏的内容。

可以看看下图，1.1M 的 icon.png 里面是否藏着什么秘密了？

![1.1M Picture](http://o8p68x17d.bkt.clouddn.com/hide_in_picture.png)

接下来在应用启动时，将其中的 icon.png 后面的 jar 包读取出来，并加载到 classLoader 中，就可以正常启动了。下面的 smali 代码，就是其他应用正在解析 icon.png 背后东西的部分代码。

```java
const-string v6, "icon.png"

invoke-virtual {v3, v6}, Landroid/content/res/AssetManager;->open(Ljava/lang/String;)Ljava/io/InputStream;

move-result-object v16

const-wide/16 v6, ----

move-object/from16 v0, v16

invoke-virtual {v0, v6, v7}, Ljava/io/InputStream;->skip(J)J
```

可能有人好奇，我是怎么想到这种方案的？嘿嘿，不告诉你。

即便使用了这种方法，也不能保定周全，有些开发者可以直接进行二次打包。而且反编译应用包后，修改资源再二次打包，并发布出去这样的做法，已经形成一个灰色产业链，特别是对国外的游戏进行修改并加广告层出不穷，在这样相对恶劣的情况下，应用开发商特别是独立开发者要学会用合适的方式来保护好自身。

期待一个更好，更健康的 Android 生态圈~

------------------------

### 参考文章

1. [Android APK打包流程](http://shinelw.com/2016/04/27/android-make-apk/)
2. [Configure Your Build](https://developer.android.com/studio/build/index.html)
3. [Android 命令行手动编译打包详解](http://jojol-zhou.iteye.com/blog/729254)

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年7月7日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
