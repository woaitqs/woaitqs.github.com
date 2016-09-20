---
layout: post
title: "大话插件 - 解析 APK 信息"
description: "Android ClassLoader 相关的原理，如何用动态解析 APK 信息"
keywords: "android 插件 APK 源码解析 "
category: "android"
tags: [android, program, plugin]
---
{% include JB/setup %}

在上一篇文章 [Android ClassLoader 加载机制](http://www.woaitqs.cc/android/2016/07/26/android-plugin-dynamic-load-classes.html) 简单介绍了 Android 的类加载机制，以及我们如何通过这个机制，加载位于 SD 中的APK，并通过同一接口的方式，来实现宿主程序和插件程序之间的解耦。接下来在这篇文章中，将说明如何获取插件 APK 的 Activity, Service 等组件信息，如何将这些内容据为己用。

在具体讲解前，最好先阅读下 [Android 应用安装过程源码解析](http://www.woaitqs.cc/android/2016/07/28/android-plugin-get-apk-info.html), 这里详细说明了 Android 是如何安装一个应用，对这一过程的了解，有助于我们在此基础上，通过反射、代理等方式，绕过系统限制，达成我们不通过系统安装过程的情况下，加载对应的 APK 文件。

<!--break-->

### 解析 APK 信息

Android 对 APK 信息的解析主要围绕 PackageParser 来进行，其负责解析 APK 中包含的信息。PackageParser 源码支持单个 APK ，也支持多个 APK 组合起来的情况。既然我们的目的是获取 APK 相关的信息，那直接调用不就行了吗？PackageParser 这个类使用了 @hide 注解, 使得我们无法在编译的时候申明对这个类的使用。Google 官方是这么申明的:

> When applied to a package, class, method or field, @hide removes that node and all of its children from the documentation.

换而言之，申明了 hide 注解，就意味着在编译时无法使用它，尽管在实际的运行环境中其是存在的。这样既能帮助类库实现类直接的调用，也能将这些信息做到对外部屏蔽。既然不能直接使用 PackageParser, 那么如何又该如何绕过这个限制呢。

主流的方法，是如下两种：

1. 通过反射调用。由于 PackageParser 大多都是 static 的方法，通过反射的方式调用，就可以绕过 hide 注解的限制。

```java
Class<?> packageParserClass = Class.forName("android.content.pm.PackageParser")
```

但这个类的兼容性非常的差，也许是因为其 @hide 的注解申明，导致不同 SDK 下其方法的签名不竟相同，这样就使得我们必须对这个类在不同 SDK 版本下进行相应的兼容处理。360 的 [DroidPlugin](https://github.com/DroidPluginTeam/DroidPlugin) 的方案就是基于此的方案, 查看其代码能够看到有 PackageParserApi15, PackageParserApi16, PackageParserApi21Preview1 等等好几个文件，360 使用这套方案时，也是对兼容性做了充分的处理。360 的 DroidPlugin 方案抽象了 PackageParser ，通过这个抽象的 PackageParser 屏蔽了不同的细节。

下面的代码，是 360 针对 SDK 22 版本进行的兼容处理。

```java
@Override
public PackageInfo generatePackageInfo(
        int gids[], int flags, long firstInstallTime, long lastUpdateTime,
        HashSet<String> grantedPermissions) throws Exception {
    try {
        return super.generatePackageInfo(gids, flags, firstInstallTime, lastUpdateTime, grantedPermissions);
    } catch (Exception e) {
        Log.i(TAG, "generatePackageInfo fail", e);
    }

    Method method = MethodUtils.getAccessibleMethod(sPackageParserClass, "generatePackageInfo",
            mPackage.getClass(),
            int[].class, int.class, long.class, long.class, sArraySetClass, sPackageUserStateClass, int.class);
    Object grantedPermissionsArray = null;
    try {
        Constructor constructor = sArraySetClass.getConstructor(Collection.class);
        grantedPermissionsArray = constructor.newInstance(constructor, grantedPermissions);
    } catch (Exception e) {
        e.printStackTrace();
    }
    if (grantedPermissionsArray == null) {
        grantedPermissionsArray = grantedPermissions;
    }
    return (PackageInfo) method.invoke(null, mPackage, gids, flags, firstInstallTime, lastUpdateTime, grantedPermissionsArray, mDefaultPackageUserState, mUserId);
}
```

2. 上一种方案的确定就是需要进行相对复杂的兼容性处理，在维护的时候会有一定的困难。为了解决这个问题，也就有了第二种方案，就是通过自定义 PackageParser 的方式，来避免在各个版本的签名兼容代码。[VirtualApp](https://github.com/asLody/VirtualApp) 就采用了这样的实现方案，具体实现在[这里](https://github.com/asLody/VirtualApp/blob/master/VirtualApp/lib/src/main/java/com/lody/virtual/service/pm/PackageParser.java)。

VirtualApp 在解析 APK 信息时，抽象了如下的接口。

```java
public Package parsePackage(File sourceFile, String destCodePath, DisplayMetrics metrics, int flags)
```

其中 sourceFile 是指 APK 文件，destCodePath 是指解压的位置，metrics 用于表明 Display 特征，flags 用于指定特定的操作。

```java
// AssetManager 管理相应的资源，并使其能被访问到。
assmgr = new AssetManager();
int cookie = assmgr.addAssetPath(mArchiveSourcePath);
if (cookie != 0) {
  res = new Resources(assmgr, metrics, null);
  assmgr.setConfiguration(0, 0, null, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
      Build.VERSION.RESOURCES_SDK_INT);
  parser = assmgr.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);
  assetError = false;
} else {
  VLog.w(TAG, "Failed adding asset path:" + mArchiveSourcePath);
}
```

```java
try {
  pkg = parsePackage(res, parser, flags, errorText);
} catch (Exception e) {
  errorException = e;
  mParseError = PackageManager.INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION;
}
```

如果不出意外，就能得到相应的 Package 信息，包含其中的 Activity 列表，Receiver 列表等等，也包含请求的权限，安装位置等等。

### 将 APK 放置到自己的目录下

Android 系统是基于 Linux 系统的，其权限系统也是类似的。Android 系统将每个应用当做一个用户，每个应用只能访问自己目录下的内容，如果想要在未安装的情况下运行相应的 APK，那么就只能将其放在自己的目录下。

1. 拷贝 lib 文件

```java
public static int copyNativeBinaries(File apkFile, File sharedLibraryDir) {
  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    return copyNativeBinariesAfterL(apkFile, sharedLibraryDir);
  } else {
    return copyNativeBinariesBeforeL(apkFile, sharedLibraryDir);
  }
}
```

2. 拷贝 APK 文件
拷贝到 /data/data/io.virtualapp/virtualApp/{packagename}/apk/base.apk

3. 创建相应的目录

dvm cache: /data/data/io.virtualapp/virtualApp/{packagename}/dalvik-cache

cache: /data/data/io.virtualapp/virtualApp/{packagename}/cache

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年8月11日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
