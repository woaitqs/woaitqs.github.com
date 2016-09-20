---
layout: post
title: "详细了解Android Context"
description: "Android context 原理解析"
keywords: "android context 原理解析"
category: "android"
tags: [android, context, program]
---
{% include JB/setup %}

Context 对于开发人员实在太常见了，各种 API 调用都需要 Context 的参与，如此广泛地出现，那就很有必要进行下深入地学习和理解，避免错误用法导致的内存泄露等等问题。

<!--break-->

------------------------------------------

### 0X00 Context 是什么

在了解 Context 之前，看看为啥要有它。Context 的中文意思是上下文，提供了获取上下文相关信息的接口。通过 Context 能够获得应用级别的信息，这些信息包括 PackageManager、 Resource、 uid、System Service 等等。而这些信息，对于各种操作都是非常有必要的，我们看看 Toast 的 makeText 方法。

```java
public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
    Toast result = new Toast(context);

    LayoutInflater inflate = (LayoutInflater)
            context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
    TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
    tv.setText(text);

    result.mNextView = v;
    result.mDuration = duration;

    return result;
}
```

在 makeText 方法里面，通过 context 得到 LAYOUT_INFLATER_SERVICE 的系统服务。在其他需要 Context 的地方，也用着 Context 提供的各种信息。总结起来就是， Context 作为上下文环境信息的提供者，类似于一个工具集。

------------------------------------------

### 0X01 Context 架构

作为一个上下文环境，该如何实现呢？接下来直接明了地，看看 Context 具体的设计图。

![图片来自 http://www.cnblogs.com/goodhacker/p/4241466.html](http://o8p68x17d.bkt.clouddn.com/context.png)

Context 本身是一个抽象类，声明了作为一个 Context 对象需要实现的方法，也就是需要提供的上下文信息，例如 `getResources`,`getPackageManager` 等等。

```java
public abstract class Context {

  /** Return an AssetManager instance for your application's package. */
  public abstract AssetManager getAssets();

  /** Return a Resources instance for your application's package. */
  public abstract Resources getResources();

  /** Return PackageManager instance to find global package information. */
  public abstract PackageManager getPackageManager();

  /** Return a ContentResolver instance for your application's package. */
  public abstract ContentResolver getContentResolver();

  // other codes...
}
```

ContextWrapper 是其中的一个子类，这个是代理的实现，这种代理模式在 Binder 机制中也被使用，机制是相同的。通过构造函数，或者 attachBaseContext 函数传入 baseContext，而在实际的方法中，就通过这个 baseContext 来进行实际的操作。下面代码中的 `getResources` 方法，演示了如何通过 baseContext 进行的调用。

```java
public class ContextWrapper extends Context {

    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }

    /**
     * Set the base context for this ContextWrapper.  All calls will then be
     * delegated to the base context.  Throws
     * IllegalStateException if a base context has already been set.
     *
     * @param base The new base context for this wrapper.
     */
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }

    @Override
    public Resources getResources(){
        return mBase.getResources();
    }
}
```
ContextImpl 是主要的实现类，实现了 Context 的大部分方法，针对 Application、Activity、Service 提供了对应的构造方法。

```java
class ContextImpl extends Context {

  // other codes...

  static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
    return new ContextImpl(null, mainThread,
            packageInfo, null, null, false, null, null, Display.INVALID_DISPLAY);
  }

  static ContextImpl createActivityContext(ActivityThread mainThread,
          LoadedApk packageInfo, int displayId, Configuration overrideConfiguration) {
      if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
      return new ContextImpl(null, mainThread, packageInfo, null, null, false,
              null, overrideConfiguration, displayId);
  }


  @Override
  public PackageManager getPackageManager() {
      if (mPackageManager != null) {
          return mPackageManager;
      }

      IPackageManager pm = ActivityThread.getPackageManager();
      if (pm != null) {
          // Doesn't matter if we make more than one instance.
          return (mPackageManager = new ApplicationPackageManager(this, pm));
      }

      return null;
  }
}
```

ContextThemeWrapper 是在 ContextWrapper 的基础上，进行了主题相关的封装。因而 Activty 会继承这个，实现了主题相关的逻辑。而另一方面 Service 没有和主题相关的内容，这里继承自 ContextWrapper。

```java
public class ContextThemeWrapper extends ContextWrapper {

  public ContextThemeWrapper() {
      super(null);
  }

  public ContextThemeWrapper(Context base, @StyleRes int themeResId) {
      super(base);
      mThemeResource = themeResId;
  }

  public ContextThemeWrapper(Context base, Resources.Theme theme) {
      super(base);
      mTheme = theme;
  }
}
```

### 0X02 Context 生命周期

根据前面的架构图，可以看到分别有 Application Context、Activity Context 和 Service Context 三种，接下来分析下他们的构建和销毁时间，这样可以帮助我们判断什么时候该用什么 Context。

#### Application Context

想看 Application Context 的创建时间，很自然地就会想到先看看 Application 的构建时候，也就是 ActivityThread 中的 handleBindApplication 方法。

```java
private void handleBindApplication(AppBindData data) {

  final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);

  try {
    java.lang.ClassLoader cl = instrContext.getClassLoader();
    mInstrumentation = (Instrumentation)
        cl.loadClass(data.instrumentationName.getClassName()).newInstance();
  } catch (Exception e) {
      throw new RuntimeException(
          "Unable to instantiate instrumentation "
          + data.instrumentationName + ": " + e.toString(), e);
  }

  mInstrumentation.init(this, instrContext, appContext,
         new ComponentName(ii.packageName, ii.name), data.instrumentationWatcher,
         data.instrumentationUiAutomationConnection);

  try {
       // If the app is being launched for full backup or restore, bring it up in
       // a restricted environment with the base application class.
       Application app = data.info.makeApplication(data.restrictedBackupMode, null);
       mInitialApplication = app;

       // Do this after providers, since instrumentation tests generally start their
       // test thread at this point, and we don't want that racing.
       try {
           mInstrumentation.onCreate(data.instrumentationArgs);
       }

       try {
           mInstrumentation.callApplicationOnCreate(app);
       } catch (Exception e) {
           if (!mInstrumentation.onException(app, e)) {
               throw new RuntimeException(
                   "Unable to create application " + app.getClass().getName()
                   + ": " + e.toString(), e);
           }
       }
   } finally {
       StrictMode.setThreadPolicy(savedPolicy);
   }

}
```

在 LoadedApk 中的 makeApplication 方法中，调用了 Instrumentation 的 newApplication 方法。

```java
try {
    java.lang.ClassLoader cl = getClassLoader();
    if (!mPackageName.equals("android")) {
        initializeJavaContextClassLoader();
    }
    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
    app = mActivityThread.mInstrumentation.newApplication(
            cl, appClass, appContext);
    appContext.setOuterContext(app);
} catch (Exception e) {
    if (!mActivityThread.mInstrumentation.onException(app, e)) {
        throw new RuntimeException(
            "Unable to instantiate application " + appClass
            + ": " + e.toString(), e);
    }
}
```

Instrumentation 中的 newApplication 方法，调用了 Application 中的 attachBaseContext 方法，从这一步开始，Application Context 就暴露出去，并可以在应用中使用了。

```java
static public Application newApplication(Class<?> clazz, Context context)
        throws InstantiationException, IllegalAccessException,
        ClassNotFoundException {
    Application app = (Application)clazz.newInstance();
    app.attach(context);
    return app;
}

/* package */ final void attach(Context context) {
    attachBaseContext(context);
    mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
}

```

这个 Context 的生命周期和 Application 一致，一般情况下使用这个 Context 不会引发内存泄露等问题。

#### Activity Context

Activity 在启动的时候会进行 context 的创建，相关的代码在 ActivityThread 中的 perfromLaunchActivity 方法。

```java

Activity activity = null;
try {
    java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
    activity = mInstrumentation.newActivity(
            cl, component.getClassName(), r.intent);
    StrictMode.incrementExpectedActivityCount(activity.getClass());
    r.intent.setExtrasClassLoader(cl);
    r.intent.prepareToEnterProcess();
    if (r.state != null) {
        r.state.setClassLoader(cl);
    }
} catch (Exception e) {
    if (!mInstrumentation.onException(activity, e)) {
        throw new RuntimeException(
            "Unable to instantiate activity " + component
            + ": " + e.toString(), e);
    }
}

if (activity != null) {
      Context appContext = createBaseContextForActivity(r, activity);
      CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
      Configuration config = new Configuration(mCompatConfiguration);
      if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
              + r.activityInfo.name + " with config " + config);
      activity.attach(appContext, this, getInstrumentation(), r.token,
              r.ident, app, r.intent, r.activityInfo, title, r.parent,
              r.embeddedID, r.lastNonConfigurationInstances, config,
              r.referrer, r.voiceInteractor);

      // other codes.

      activity.mCalled = false;
      if (r.isPersistable()) {
          mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
      } else {
          mInstrumentation.callActivityOnCreate(activity, r.state);
      }

}
```

在 callActivityOnCreate 方法里，会执行 Activity 的 onCreate 方法，从这开始 Activity 这个 Context 就已经可用了。这里可以看到 Activity 与 新建的 context 紧紧地绑定在一起。如果我们在一些耗时操作中，引用了 Activity 这种 Context 就可能会导致内存泄露。

#### Service Context

```java
private void handleCreateService(CreateServiceData data) {
    // ......
    Service service = null;
    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        service = (Service) cl.loadClass(data.info.name).newInstance();
    } catch (Exception e) {
        ......
    }

    try {
        // ......

        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);

        Application app = packageInfo.makeApplication(false, mInstrumentation);
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManagerNative.getDefault());
        service.onCreate();
        // ......
    } catch (Exception e) {
        // ......
    }
}
```

Service 与 Activity 类似，也是与 Service 同样生命周期。

### 0X04 各种 Context 的使用场景

![图片来自 http://www.jianshu.com/p/94e0f9ab3f1d ](http://o8p68x17d.bkt.clouddn.com/context_usage.png)

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年9月9日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
