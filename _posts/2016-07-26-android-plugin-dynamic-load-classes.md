---
layout: post
title: "大话插件 - ClassLoader 加载机制"
description: "Android ClassLoader 相关的原理"
keywords: "android 插件 lifecycle 源码解析 binder activity"
category: "android"
tags: [android, program, plugin]
---
{% include JB/setup %}

Java 代码在编译过后，会生成相应的 Class 文件，在实际执行的时候，Java 虚拟机（JVM）会实际运行相应的 Class 文件。对于 Davlik 虚拟机和 ART 虚拟机也是类似的机制。如果要通过插件的形式来执行插件中的逻辑，因为我们需要动态地加载插件中的 class 文件，巧妇难为无米之炊，就是这个道理，因而这篇文章的目的，就是了解 Android 的类加载机制，以及通过实际的例子来看看如何运用这个来达到我们加载插件的目的。

<!--break-->

--------------------------------

### ClassLoader 机制

JVM 可以在运行时加载 Class ，而无需知道起具体的信息，这种机制就是 Java 的类加载机制，也称为 ClassLoader 机制。在 Java 中类加载器非常重要，我们所说的 equals、instanceOf 等方法的应用都局限在同一个类加载器中，类似于命名空间的概念，换而言之，两个同样的类如果在不同的加载器中，那么两个类就不会相等。

Java 中一共拥有 3 种类型的类加载：

> 1. 启动（Bootstrap）类加载器：是用本地代码实现的类装入器，它负责将 <Java_Runtime_Home>/lib下面的类库加载到内存中（比如rt.jar）。由于引导类加载器涉及到虚拟机本地实现细节，开发者无法直接获取到启动类加载器的引用，所以不允许直接通过引用进行操作。
> 2. 标准扩展（Extension）类加载器：是由 Sun 的 ExtClassLoader（sun.misc.Launcher$ExtClassLoader）实现的。它负责将< Java_Runtime_Home >/lib/ext或者由系统变量 java.ext.dir指定位置中的类库加载到内存中。开发者可以直接使用标准扩展类加载器。
> 3. 系统（System）类加载器：是由 Sun 的 AppClassLoader（sun.misc.Launcher$AppClassLoader）实现的。它负责将系统类路径（CLASSPATH）中指定的类库加载到内存中。开发者可以直接使用系统类加载器。

先看看，我们通常会使用的几种 ClassLoader，如下的代码所示，运行下看看结果。

```Java
ClassLoader classLoader = getClassLoader();
Log.d("class_loader", classLoader.toString());
while (classLoader.getParent() != null) {
  classLoader = classLoader.getParent();
  Log.d("class_loader", classLoader.toString());
}
```

```shell
D/class_loader: dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.plugin.android.learn_class_loader-1/base.apk"],nativeLibraryDirectories=[/data/app/com.plugin.android.learn_class_loader-1/lib/arm, /vendor/lib, /system/lib]]]
D/class_loader: java.lang.BootClassLoader@fa91c1
```

大家可以看到这里调用了 `getParent` 这个方法，是的，ClassLoader 是可以拥有父亲节点的，JVM 规定除了 BootClassLoader (根节点)以外的所有 ClassLoader 都必须拥有父亲节点。这样设计的好处是什么？这是 Java 的双亲委派机制。

> 某个特定的类加载器在接到加载类的请求时，首先将加载任务委托给父类加载器，依次递归，如果父类加载器可以完成类加载任务，就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载。

在这样的机制下，能够极大可能地节省加载资源，只要父亲节点加载过该类，那其他子节点就不用再进行加载了，从而在内存中就只有一份字节码，另一方面，这样的机制也可以提升安全性。当某些恶意人员企图通过在其 ClassLoader 中修改系统 Class 文件时，就会因为双亲委派机制而无法加载。

![classloader_tree](http://o8p68x17d.bkt.clouddn.com/classloader_tree.jpg)

更多关于 ClassLoader 的资料，请参考这个链接 [深入探讨 Java 类加载器](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/)

--------------------------------

### Android 中的 ClassLoader 机制

尽管在实现上与 JVM 大同小异，但还是有不少特殊的地方。运行的 Android 虚拟机是 Davlik 或者 ART，具体执行的也不是 Class 文件，而是针对移动设备进行优化后的 DEX 文件。当然 Android 系统也提供了类似的加载机制，主要是由 BaseDexClassLoader 的两个子类 `PathClassLoader` 和 `DexClassLoader` 来完成。

先看看 `DexClassLoader` 的使用方法。

```java
public class DexClassLoader extends BaseDexClassLoader {
    /**
     * Creates a {@code DexClassLoader} that finds interpreted and native
     * code.  Interpreted classes are found in a set of DEX files contained
     * in Jar or APK files.
     *
     * <p>The path lists are separated using the character specified by the
     * {@code path.separator} system property, which defaults to {@code :}.
     *
     * @param dexPath the list of jar/apk files containing classes and
     *     resources, delimited by {@code File.pathSeparator}, which
     *     defaults to {@code ":"} on Android
     * @param optimizedDirectory directory where optimized dex files
     *     should be written; must not be {@code null}
     * @param libraryPath the list of directories containing native
     *     libraries, delimited by {@code File.pathSeparator}; may be
     *     {@code null}
     * @param parent the parent class loader
     */
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}
```

如上面的注释说述，可以通过这 4 个参数来构造自己的 ClassLoader，dexPath 是指 dex 文件的位置，optimizedDirectory 指的是在解压后，生成的文件放置的位置，这里出于安全性的考虑，只能放在自己的目录下，另一方面 libraryPath 是指 library 存放的目录，最后 parent 是指父 ClassLoader，我们前面提过，任何一个 ClassLoader 除了 BootClassLoader 以外都必须含有父亲节点。

再来看看 `PathClassLoader` 的构造函数。

```java
/**
 * Provides a simple {@link ClassLoader} implementation that operates on a list
 * of files and directories in the local file system, but does not attempt to
 * load classes from the network. Android uses this class for its system class
 * loader and for its application class loader(s).
 */
public class PathClassLoader extends BaseDexClassLoader {
    /**
     * Creates a {@code PathClassLoader} that operates on a given list of files
     * and directories. This method is equivalent to calling
     * {@link #PathClassLoader(String, String, ClassLoader)} with a
     * {@code null} value for the second argument (see description there).
     *
     * @param dexPath the list of jar/apk files containing classes and
     * resources, delimited by {@code File.pathSeparator}, which
     * defaults to {@code ":"} on Android
     * @param parent the parent class loader
     */
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
    /**
     * Creates a {@code PathClassLoader} that operates on two given
     * lists of files and directories. The entries of the first list
     * should be one of the following:
     *
     * <ul>
     * <li>JAR/ZIP/APK files, possibly containing a "classes.dex" file as
     * well as arbitrary resources.
     * <li>Raw ".dex" files (not inside a zip file).
     * </ul>
     *
     * The entries of the second list should be directories containing
     * native library files.
     *
     * @param dexPath the list of jar/apk files containing classes and
     * resources, delimited by {@code File.pathSeparator}, which
     * defaults to {@code ":"} on Android
     * @param libraryPath the list of directories containing native
     * libraries, delimited by {@code File.pathSeparator}; may be
     * {@code null}
     * @param parent the parent class loader
     */
    public PathClassLoader(String dexPath, String libraryPath,
            ClassLoader parent) {
        super(dexPath, null, libraryPath, parent);
    }
}
```

这里的参数和前面 DexClassLoader 的含义是一样的，但是没有 optimizedDirectory 这个参数。PathClassLoader 在实现的时候，就是为系统应用服务的，在使用的时候，会从 /data/dalvik-cache/ 去查找，如果应用未安装，那么会触发异常。这也就限定了，这个类只试用于已安装的应用，插件化方案在一般情况下不会使用这个。

### Android 应用实例

假定现在有一个需求，是将二维码扫描功能从主线上剥离出去，抽成单独的插件。这里为了简单起见，将二维码扫描逻辑简化如下：

```java
public class QrTask {

  public static String scanQrImage() {
    return "Haha, the website is www.woaitqs.cc";
  }

}
```

我们将上面的内容打包成APK(jar 文件也可以，放置在 Assert 目录中去)，放到 sdcard 目录下，下面我们来通过 DexClassLoader 的方式动态地调用这里面的代码。

![apk_in_sdcard](http://o8p68x17d.bkt.clouddn.com/apk_in_sdcard.png)

```java
public void callInPlugin() {
  String apkPath = Environment.getExternalStorageDirectory().getAbsolutePath()
      + "/target_final.jar";
  File dexFile =  getDir("dex", Context.MODE_PRIVATE);
  DexClassLoader classLoader = new DexClassLoader(
      apkPath, dexFile.getAbsolutePath(), null, getClassLoader());
  try {
    Class<?> pluginClass = classLoader.loadClass("com.plugin.learn_class_loader.QrTask");
    Object object = pluginClass.newInstance();
    Method method = pluginClass.getDeclaredMethod("scanQrImage", null);
    Log.d("class_loader", method.invoke(object, null).toString());
  } catch (ClassNotFoundException e) {
    e.printStackTrace();
  } catch (InstantiationException e) {
    e.printStackTrace();
  } catch (IllegalAccessException e) {
    e.printStackTrace();
  } catch (NoSuchMethodException e) {
    e.printStackTrace();
  } catch (InvocationTargetException e) {
    e.printStackTrace();
  }
}
```

代码很简单，这里用的了一些 Android 反射的知识，如果对此有不了解的话，可以看看这个链接 [Android 极简反射教程，及应用示例](http://www.woaitqs.cc/android/2016/07/14/android-reflection.html)。在代码中可以看到，这里通过 DexClassLoader 的方式来加载 QrTask 这个类，并最后通过反射的方式，调用了其中的 scanQrImage 方法。

--------------------------------

### 复用性更高的方案

在上面的例子中，由于在宿主工程中没有插件工程的代码，所以采用了反射的调用方式，但这在可维护性上和可读性上都大打折扣，现在来寻求一种方式使得既能编译，也在可维护性上也达到一个较好的效果。

既然在宿主程序中，没有插件的代码，那么可以通过接口的方式来做一次中介。一般情况下，将声明的插件放置在单独的工程中，插件工程和宿主工程都依赖于这个工程，从而解决编译的问题。这样在宿主工程中使用这个接口即可，来看看具体的例子。

```java
public interface IQrTask {

  String scanQrImage();

}

public class QrTask implements IQrTask{

  public String scanQrImage() {
    return "Haha, the website is www.woaitqs.cc";
  }

}
```

在宿主工程中，通过针对接口编程，使得代码能够清晰不少。

```java
Class<?> pluginClass = classLoader.loadClass("tango.QrTask");
IQrTask task = (IQrTask) pluginClass.newInstance();
Log.d("class_loader", task.scanQrImage());
```

在实际应用中，通过接口的插件方案在使用的时候，还有一些需要注意的地方。这些技术已经发展得比较完善，具体可以看这个链接，[Android动态加载技术三个关键问题详解
](https://blog.tingyun.com/web/article/detail/166)。

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年7月26日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
