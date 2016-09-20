---
layout: post
title: "大话插件 - 动态加载插件 Activity"
description: "Android ClassLoader 相关的原理，动态加载 Activity 的技术"
keywords: "android 插件 lifecycle 源码解析 binder activity"
category: "android"
tags: [android, program, plugin]
---
{% include JB/setup %}

有时候稍不注意, 忘记在 Manifest 文件中注册 Activity，在运行的时候启动 Activity 时就会触发 `ActivityNotFoundException` 的异常。对于每一个运行的 Activity 都需要进行注册，这个常识我们都很清楚，但是在插件中这样的要求就有些难以实现，由于宿主程序在设计的时候，不知道插件的细节，更不用说在宿主程序的 Manifest 里面提前注册插件 Activity。

在这篇文章中，介绍了几种可以绕过 Android 对 Activity 需要注册的限制的实现方式。对这些实现方式的了解，有助于理解 Activity 背后的原理，加深对 ActivityManagerService 等等重要系统服务的认知，是不错的进阶知识。

在正式开始写之前，我还是想额外地扯扯淡。就我自身看来，插件化技术本身的未来是不明朗的，在后续日趋稳定的类 Reactive Native 技术稳定（国内有 Weex）后，可以帮助我们屏蔽不同版本的兼容性问题，实现动态功能的成本也更低，可能更适合于长远方向。但我依旧还在学习插件化技术，是在于插件化技术的深入理解需要依托于对 Android Framework 层的透彻了解上，通过对此的学习，对自身内功的修炼很有裨益。Android 技术也日新月异的发展，而背后的 Framework 层则相对稳定，设计理念也是大体相同，对于 Framework 层的理解能帮我们构建出更好的程序。这就是你所不能被其他人替代的地方，因为你的不可替代性，也能赢得更好的机会。

<!--break-->

### 利用接口伪装

[dynamic-load-apk](https://github.com/singwhatiwanna/dynamic-load-apk) 作为第一个将 Android 插件化开源方案出去的项目，提供了最初的解决方案，

#### Manifest 注入代理 ProxyActivity

```xml
<activity
    android:name="com.ryg.dynamicload.DLProxyActivity"
    android:label="@string/app_name" >
    <intent-filter>
        <action android:name="com.ryg.dynamicload.proxy.activity.VIEW" />

        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

既然不能提前在 Manifest 里面注册相应的 Activity ，那么就提前注册代理 ProxyActivity，这个代理 Activity 在启动后，会通过静态代理的方式，再实际调用真实 Activity 的方法。

这里一定要在宿主程序中，声明 DLProxyActivity，目前还没有什么方案可以绕过不需要声明的限制。

#### 启动插件 Activity

通常启动 Activity 的时候，代码是这样实现的。

```java
Intent intent = new Intent(context, TargetActivity.class);
context.startActivity(intent);
```

[dynamic-load-apk](https://github.com/singwhatiwanna/dynamic-load-apk) 在实现的时候为了实现自己代理的效果，进行了自己的封装，将 Intent 封装成 DLIntent。如下面的代码所示，DLIntent 将插件包名和对应插件的 Activity 类传递进来。

```java
public DLIntent(String pluginPackage, String pluginClass) {
    super();
    this.mPluginPackage = pluginPackage;
    this.mPluginClass = pluginClass;
}

public DLIntent(String pluginPackage, Class<?> clazz) {
    super();
    this.mPluginPackage = pluginPackage;
    this.mPluginClass = clazz.getName();
}
```

接下来看看 [dynamic-load-apk](https://github.com/singwhatiwanna/dynamic-load-apk) 如何实现启动 startActivity 的。

```java
public int startPluginActivityForResult(Context context, DLIntent dlIntent, int requestCode) {

    String packageName = dlIntent.getPluginPackage();
    //验证intent的包名
    if (TextUtils.isEmpty(packageName)) {
        throw new NullPointerException("disallow null packageName.");
    }
    //检测插件是否加载
    DLPluginPackage pluginPackage = mPackagesHolder.get(packageName);
    if (pluginPackage == null) {
        return START_RESULT_NO_PKG;
    }
    //要调用的插件Activity的class完整路径
    final String className = getPluginActivityFullPath(dlIntent, pluginPackage);
    //Class.forName
    Class<?> clazz = loadPluginClass(pluginPackage.classLoader, className);
    if (clazz == null) {
        return START_RESULT_NO_CLASS;
    }
    //获取代理Activity的class，DLProxyActivity/DLProxyFragmentActivity
    Class<? extends Activity> proxyActivityClass = getProxyActivityClass(clazz);
    if (proxyActivityClass == null) {
        return START_RESULT_TYPE_ERROR;
    }
    //put extra data
    dlIntent.putExtra(DLConstants.EXTRA_CLASS, className);
    dlIntent.putExtra(DLConstants.EXTRA_PACKAGE, packageName);
    dlIntent.setClass(mContext, proxyActivityClass);
    //通过context启动宿主Activity
    performStartActivityForResult(context, dlIntent, requestCode);
    return START_RESULT_SUCCESS;
}
```

首先通过 ClassLoader 的方式(这里有具体介绍 [Android ClassLoader 加载机制](http://www.woaitqs.cc/android/2016/07/26/android-plugin-dynamic-load-classes.html)) 加载插件。同样在 DLIntent 里面将实际的插件包名和插件对象传递进去，以便后续代理 Activity 调用。注意这里的 `getProxyActivityClass` 方法返回的是 `DLProxyActivity`, 也就是说将要启动的 Activity 替换为了代理 Activity。

#### 处理插件生命周期

当 ProxyActivity 启动后，在相应的生命周期时通过反射的方式调用实际 Activity 中生命周期的方法，但反射这种方式存在两个方法的问题，一是频繁地调用反射会有不可忽视的性能开销，另一方面反射的使用会使得代码难以维护，而且可能存在兼容性问题。就先定义了如下的接口：

```java
public interface DLPlugin {

    public void onCreate(Bundle savedInstanceState);
    public void onStart();
    public void onRestart();
    public void onActivityResult(int requestCode, int resultCode, Intent data);
    public void onResume();
    public void onPause();
    public void onStop();
    public void onDestroy();
    public void attach(Activity proxyActivity, DLPluginPackage pluginPackage);
    public void onSaveInstanceState(Bundle outState);
    public void onNewIntent(Intent intent);
    public void onRestoreInstanceState(Bundle savedInstanceState);
    public boolean onTouchEvent(MotionEvent event);
    public boolean onKeyUp(int keyCode, KeyEvent event);
    public void onWindowAttributesChanged(LayoutParams params);
    public void onWindowFocusChanged(boolean hasFocus);
    public void onBackPressed();
    public boolean onCreateOptionsMenu(Menu menu);
    public boolean onOptionsItemSelected(MenuItem item);

}
```

代理 Activity 实现了这个接口，并在相应的接口中，去调用插件 Activity 的方法，从而实现偷天换日的功效。下面以 `finish` 函数为例，说明如何实现的。

```java
public class DLBasePluginActivity extends Activity implements DLPlugin {

  // ...

  @Override
  public void attach(Activity proxyActivity, DLPluginPackage pluginPackage) {
      mProxyActivity = (Activity) proxyActivity;
      that = mProxyActivity;
      mPluginPackage = pluginPackage;
  }

  @Override
  public void finish() {
      if (mFrom == DLConstants.FROM_INTERNAL) {
          super.finish();
      } else {
          mProxyActivity.finish();
      }
  }

  // ...

}
```

这种方案只差最后，怎么将 TargetActivity 和 ProxyActivity 绑定在一起了? dynamic-load-apk 中 launchTargetActivity 实现了这个功能，在其中的 attach 函数里面，将 ProxyActivity 和 PluginActivity 绑定在一起。

```java
protected void launchTargetActivity() {
    try {
        Class<?> localClass = getClassLoader().loadClass(mClass);
        Constructor<?> localConstructor = localClass.getConstructor(new Class[] {});
        Object instance = localConstructor.newInstance(new Object[] {});
        mPluginActivity = (DLPlugin) instance;
        ((DLAttachable) mProxyActivity).attach(mPluginActivity, mPluginManager);
        // attach the proxy activity and plugin package to the mPluginActivity
        mPluginActivity.attach(mProxyActivity, mPluginPackage);

        Bundle bundle = new Bundle();
        bundle.putInt(DLConstants.FROM, DLConstants.FROM_EXTERNAL);
        mPluginActivity.onCreate(bundle);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

经过上诉的步骤，就可以绕过 Activity 需要注册的限制了，但这个方案也有一定的限制。不仅要求，插件和宿主都必须同时依赖于一个接口工程，这样会严重制约插件的实现。另一方面，对于 Activity 可以这么实现，但是对于 Service 等等其他组件，也需要进行同样的接口代理，在代码的可读性上不是很好。于是，插件化在发展一段时间后，有了如下的解决方案。

------------------------------------

### 构建 APP 虚拟运行环境

构建虚拟运行环境的方式，提供了一种一劳永逸的方案，这种方案通过反射、动态代理等技术，将 APP 需要运行的系统服务进行了特殊处理，欺上瞒下，使得 APP 能在不安装的情况下，运行起来。在这种情况下，自然而然就没有前面方案中，需要额外依赖工程的弊端，也不需要插件为了继承做什么特殊处理。而实现这种虚拟运行环境，需要大量的工程，对每个系统服务都进行特殊处理，在这里就不展开叙述了，只说明 Activity 如何在这个虚拟环境下跑起来。

#### 拦截 startActivity 请求

为了不让插件做额外的工作，我们必须对拦截 startActivity, 进行偷天换日的工作。这里用到的技术就是动态代理，关于动态代理如何实现，网上有不少的文章可以参考，不再详述。 startActivty 众多签名的方法中，最后都会进入到 ActivityManager 中去，因而对 ActivityManager 进行代理是不错的选择，而实际上 ActivityManager 所做的工作是通过 Binder 机制对 ActivityManagerService 的调用。最后的代理工作，还是要回到 ActivityManagerService 里面来。

```java
if (ActivityManagerNative.gDefault.type() == IActivityManager.class) {
  ActivityManagerNative.gDefault.set(getHookObject().getProxyObject());
} else if (ActivityManagerNative.gDefault.type() == android.util.Singleton.class) {
  Object gDefault = ActivityManagerNative.gDefault.get();
  Singleton.mInstance.set(gDefault, getHookObject().getProxyObject());
}
```

上面的代码中，ActivtyManagerNative 是 ActivityManagerService 的基类，对 ActivityManagerNative 的修改作用于 ActivityManagerService。这里针对了不同 SDK 版本做了不同的处理，总之在这个替换后，ActivityManagerService 中的 gDefault 变量已经变成我们 hook 后的对象了，`getHookObject().getProxyObject()`。

```java
HookBinder<IActivityManager> hookAMBinder = new HookBinder<IActivityManager>() {
  @Override
  protected IBinder queryBaseBinder() {
    return ServiceManager.getService(Context.ACTIVITY_SERVICE);
  }

  @Override
  protected IActivityManager createInterface(IBinder baseBinder) {
    return getHookObject().getProxyObject();
  }
};
hookAMBinder.injectService(Context.ACTIVITY_SERVICE);

public void injectService(String name) throws Throwable {
  Map<String, IBinder> sCache = mirror.android.os.ServiceManager.sCache.get();
  if (sCache != null) {
    sCache.remove(name);
    sCache.put(name, this);
  } else {
    throw new IllegalStateException("ServiceManager is invisible.");
  }
}

```

在接下来的这段代码里面，是替换 SystemServer 中存放的 ActivityManagerService 变量，这样在上面两个地方进行代理之后，已经将所有和 ActivityManagerService (以下简称 AMS) 相关的入口都进行了处理，这样当我们调用 startActivity 方法时，就能进入到我们代理的对象中。接下来看看，这个代理对象应该如何实现。

代理所完成的工作是对 AMS 进行各种改动，已达成完成启动插件 Activity 的目的，这里就只从 startActivity 这个方法入手，其他方法可以逐类旁通。

我们先看看 startActivity 的方法签名，这个函数在不同版本的签名也各不相同，下面演示的是基于 SDK-23 源码。

```java
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
        String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {

        // ...

        }
```

1. 其中 intent 参数就是我们传入的启动 Intent；

2. caller 是传入到 AMS 中的 binder, 当通知 Activity 启动或者其他事件完成的时候，就可以通过这个 Binder 对象进行通知 Activty 进行生命周期处理了；

3. resultTo 这个参数是 ActivtyRecord 中的变量，这里用来表征一个 Activity。 resultTo 本事是 Binder 对象，而 Binder 对象可以在跨进程中起到唯一标示的作用。

其余参数就不再叙述了，现在看看 startActivity 具体是怎么拦截的。Java 一般使用 InvocationHandler 来进行动态代理，代理过后会调用到 `invoke`方法 , 当 method.getName 是 startActivity 时，我们就可以进行拦截了。

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  //...
}
```

获取必要的参数，并保存下来。

```java
int intentIndex = ArrayUtils.indexOfFirst(args, Intent.class);
int resultToIndex = ArrayUtils.indexOfObject(args, IBinder.class, 2);
String resolvedType = (String) args[intentIndex + 1];

Intent targetIntent = (Intent) args[intentIndex];
targetIntent.setDataAndType(targetIntent.getData(), resolvedType);
IBinder resultTo = resultToIndex != -1 ? (IBinder) args[resultToIndex] : null;
String resultWho = null;
int requestCode = 0;
Bundle options = ArrayUtils.getFirst(args, Bundle.class);
if (resultTo != null) {
  resultWho = (String) args[resultToIndex + 1];
  requestCode = (int) args[resultToIndex + 2];
}
int userId = getUserId(targetIntent);

if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR2) {
  args[intentIndex - 1] = getHostPkg();
}
```

#### 解析 startActivity 中的 Intent

在上一篇文章中，讲解到如何不经过安装过程，解析 APK 的信息。在得到这些信息之后，VirtualApp(以下简称VA) 会将这些信息组织起来，存放在本地的包服务中。信息的组织形式，与系统的 PackageManagerService 类似，将各大组件、权限等信息保留下来，当调用到 startActivity 时，解析其中的 Intent，查看是否有相应的组件匹配对应的 Intent。

```java
ActivityInfo targetActInfo = VirtualCore.getCore().resolveActivityInfo(targetIntent, userId);
if (targetActInfo == null) {
  return method.invoke(who, args);
}
String packageName = targetActInfo.packageName;
if (!isAppPkg(packageName)) {
  return method.invoke(who, args);
}
Intent resultIntent = VActivityManager.get().startActivity(targetIntent, targetActInfo, resultTo, options, userId);
if (resultIntent == null) {
  if (resultTo != null) {
    VActivityManager.get().sendActivityResult(resultTo, resultWho, requestCode);
  }
  return 0;
}
```

上面的代码中获取到 intent 对应的 targetActInfo, 如果为空，或者没有在 VirtualApp 里面，就直接调用原有 API 的方法，否则则进入我们的拦截逻辑，看起来 `VActivityManager.get().startActivity` 是重中之重。

#### 创建应用进程

在我之前的一篇文章里，[Android 应用进程启动流程](http://www.woaitqs.cc/android/2016/06/21/activity-service.html), Android 组件的运行都是需要相应的进程的。我们在讲如何启动插件 Activity 的时候，也要处理好这个问题。单独的进程有助于进行数据隔离，当发生意外情况时，不至于影响主进程。皮之不存，毛将焉附，现在看看创建进程，让插件 Activity 可以依附。

VirtualApp 会预先在 AndroidManifest 通过 `android:process` 来预置一些进程，当有需要的时候，会查看这些进程是否存在，利用其未占用的进程给插件使用。下面截取了一段 AndroidManifest 中的代码，可以注意其中的 `android:process` 字段。

```xml
<activity
    android:name="com.lody.virtual.client.stub.StubActivity$C0"
    android:configChanges="mcc|mnc|locale|...|fontScale"
    android:process=":p0"
    android:taskAffinity="com.lody.virtual.vt"
    android:theme="@style/VATheme">
    <meta-data
        android:name="X-Identity"
        android:value="Stub-User"/>
</activity>

<activity
    android:name="com.lody.virtual.client.stub.StubActivity$C1"
    android:configChanges="mcc|mnc|locale|...|fontScale"
    android:process=":p1"
    android:taskAffinity="com.lody.virtual.vt"
    android:theme="@style/VATheme">
    <meta-data
        android:name="X-Identity"
        android:value="Stub-User"/>
</activity>
```

VActivityManagerService(以下简称VAMS) 在启动后，会遍历对应 Manifest 文件中的 Activity 和 Provider 组件，并查看其中的 processName，通过 Map 建立起 processName 和对应 Activity 和 Provider 之间的关系。

```java
PackageManager pm = context.getPackageManager();
PackageInfo packageInfo = null;
try {
  packageInfo = pm.getPackageInfo(context.getPackageName(),
      PackageManager.GET_ACTIVITIES | PackageManager.GET_PROVIDERS
      | PackageManager.GET_META_DATA);
} catch (PackageManager.NameNotFoundException e) {
  e.printStackTrace();
}

ActivityInfo[] activityInfos = packageInfo.activities;
for (ActivityInfo activityInfo : activityInfos) {
  if (isStubComponent(activityInfo)) {
    String processName = activityInfo.processName;
    stubProcessList.add(processName);
    StubInfo stubInfo = stubInfoMap.get(processName);
    if (stubInfo == null) {
      stubInfo = new StubInfo();
      stubInfo.processName = processName;
      stubInfoMap.put(processName, stubInfo);
    }
    String name = activityInfo.name;
    if (name.endsWith("_")) {
      stubInfo.dialogActivityInfos.add(activityInfo);
    } else {
      stubInfo.standardActivityInfos.add(activityInfo);
    }
  }
}
```

当需要启动进程时，就从 stubInfoMap 里面去查看是否有空闲的进程可供使用。如果存在空闲的进程，则通过前面提到的 Map，从进程名得到相应的 Stub 信息。进程的创建是相对重量级的事情，而 VA 只用了几行代码，就完成了这个事情，利用的正是 Android Provider 的机制。当 Provider 启动的时候，可以同步地启动对应的进程，具体原理可以参看 [这篇文章](http://gityuan.com/2016/07/30/content-provider/).

```java
public static Bundle call(String authority, Context context, String methodName, String arg, Bundle bundle) {
  Uri uri = Uri.parse("content://" + authority);
  ContentResolver contentResolver = context.getContentResolver();
  return contentResolver.call(uri, methodName, arg, bundle);
}
```

进程启动后，还需要将这个进程和插件具体绑定起来，使得这个进程能够当做 Application 来运行，这段逻辑也相对复杂，有兴趣的可以看看 `VClientImpl` 的实现。

#### 斗转星移绕过 Manifest 的限制

在进程创建成功后，`startProcessIfNeedLocked` 可以得到对应的 ProcessRecord 对象，这个对象中存放着相应的 Activity、Service 等等组件信息，当然也包括用于偷换概念的 Stub 信息。在进程启动后，将对应的 Stub 放置在 intent 中，并通过 Binder 机制返回给 Client 端。

```java
ProcessRecord processRecord = mService.startProcessIfNeedLocked(info.processName, userId, info.packageName);
if (processRecord == null) {
  return null;
}
StubInfo selectedStub = processRecord.stubInfo;
ActivityInfo stubActInfo = selectedStub.fetchStubActivityInfo(info);
if (stubActInfo == null) {
  return null;
}
newIntent.setClassName(stubActInfo.packageName, stubActInfo.name);
newIntent.putExtra("_VA_|_intent_", intent);
newIntent.putExtra("_VA_|_stub_activity_", stubActInfo);
newIntent.putExtra("_VA_|_target_activity_", info);
newIntent.putExtra("_VA_|_user_id_", userId);
return newIntent;
```

重点来啦，在上面的 newIntent 中，将 className 设置成为代理 Activity 的信息。在 Client 端，获取到这个 resultIntent 后，将这个值注入到 startActivity 的 intent 里面去，在 args[intentIndex] = resultIntent 这里进行的替换。而在这里进行替换过后，就可以绕过 AMS 的对 Activity 需要注册的限制了。

```java
@Override
public Object onHook(Object who, Method method, Object... args) throws Throwable {
  super.onHook(who, method, args);

  // ...

  Intent resultIntent =
    VActivityManager.get().startActivity(
      targetIntent, targetActInfo, resultTo, options, userId);
  if (resultIntent == null) {
    if (resultTo != null) {
      VActivityManager.get().sendActivityResult(resultTo, resultWho, requestCode);
    }
    return 0;
  }
  args[intentIndex] = resultIntent;
  return method.invoke(who, args);
}
```

#### 给插件 Activity 注入生命

上段代码中的 `method.invoke(who, args)`, 执行的是 SDK 中 startActivity 的逻辑。这个逻辑里面执行的是启动逻辑，在这里篇幅的限制，就不再详细说明了，大家可以看我的这篇博文，[Android Activity 生命周期是如何实现的](http://www.woaitqs.cc/android/2016/07/19/how-activity-lifecircle-work), 最终程序会执行到 ActivityThread.mH 中去，对应的消息就是 `LAUNCH_ACTIVITY`，其后执行的方法就是下面代码所描述的这样。

```java
private boolean handleLaunchActivity(Message msg) {
  Object r = msg.obj;
  // StubIntent
  Intent stubIntent = ActivityThread.ActivityClientRecord.intent.get(r);
  // TargetIntent
  Intent targetIntent = stubIntent.getParcelableExtra("_VA_|_intent_");

  ComponentName component = targetIntent.getComponent();
  String packageName = component.getPackageName();

  AppSetting appSetting = VirtualCore.getCore().findApp(packageName);
  if (appSetting == null) {
    return true;
  }
  // 从 Intent 中获取的 stub 和 target 的信息
  ActivityInfo stubActInfo = stubIntent.getParcelableExtra("_VA_|_stub_activity_");
  ActivityInfo targetActInfo = stubIntent.getParcelableExtra("_VA_|_target_activity_");

  if (stubActInfo == null || targetActInfo == null) {
    return true;
  }
  String processName = ComponentUtils.getProcessName(targetActInfo);
  // 保证 Process 已经与 Application 绑定起来
  if (!VClientImpl.getClient().isBound()) {
    int targetUser = stubIntent.getIntExtra("_VA_|_user_id_", 0);
    VActivityManager.get().ensureAppBound(processName, appSetting.packageName, targetUser);
    getH().sendMessageDelayed(Message.obtain(msg), 5);
    return false;
  }

  // 设置对应的 classLoader
  ClassLoader appClassLoader = VClientImpl.getClient().getClassLoader(targetActInfo.applicationInfo);
  targetIntent.setExtrasClassLoader(appClassLoader);
  boolean error = false;
  try {
    targetIntent.putExtra("_VA_|_stub_activity_", stubActInfo);
    targetIntent.putExtra("_VA_|_target_activity_", targetActInfo);
  } catch (Throwable e) {
    error = true;
    VLog.w(TAG, "Directly putExtra failed: %s.", e.getMessage());
  }
  if (error && Build.VERSION.SDK_INT <= Build.VERSION_CODES.KITKAT) {
    ClassLoader oldParent = getClass().getClassLoader().getParent();
    mirror.java.lang.ClassLoader.parent.set(getClass().getClassLoader(), appClassLoader);
    try {
      targetIntent.putExtra("_VA_|_stub_activity_", stubActInfo);
      targetIntent.putExtra("_VA_|_target_activity_", targetActInfo);
    } catch (Throwable e) {
      VLog.w(TAG, "Secondly putExtra failed: %s.", e.getMessage());
    }
    mirror.java.lang.ClassLoader.parent.set(getClass().getClassLoader(), oldParent);
  }
  // 反射替换其中的 intent 和 activityInfo, 将 Stub 相关的信息换成 target 相关的信息
  ActivityThread.ActivityClientRecord.intent.set(r, targetIntent);
  ActivityThread.ActivityClientRecord.activityInfo.set(r, targetActInfo);
  return true;
}
```

同样也是重点，这里将 ActivityClientRecord 中的相应信息替换为了插件 Activity，从这一步过后，对应的插件 Activity 就能通过这个 H Callback 接收到相应的生命周期回调，从这一刻开始，插件 Activity 就是有血有肉的存在了。

#### 总结 VA 的实现方式

可能大家在看前面的描述过后，如果对 AMS 这一块比较熟悉的话，就会发现所做的工作其实特别简单。第一步，就是将 startActivity 中的 intent 参数，替换为插件 Activity 的信息；第二步，是在欺骗完系统后，在 H Callback 的 `LAUNCH_ACTIVITY` 消息中，将对应 Record 中的信息，还原为插件的 Activity 信息。

读者也许会问，难道真的就这么简单，就可以欺骗系统了吗？我们先通过 `adb shell dumpsys activity` 的方式，看看在 AMS 这个视角上，运行的是哪个 Activity？

![Activity Dump Trace](http://o8p68x17d.bkt.clouddn.com/stub_activity.png)

看来，AMS 还真天真地运行着 StubActivity，在前面欺骗的环节中，传递进去的确实是 StubActivity，而为何在实际运行的时候，客户端还能继续使用插件 Activity 了？在[Android Activity 生命周期是如何实现的](http://www.woaitqs.cc/android/2016/07/19/how-activity-lifecircle-work) 这篇文章里面讲到，在 ActivityThread 中的 performLaunchActivity 方法里面，实际去调用 Activity 的 onCreate 方法，而在这个 performLaunchActivity 里面有这样一段代码。

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
  ActivityInfo aInfo = r.activityInfo;

  // other code.

  activity.attach(appContext, this, getInstrumentation(), r.token,
                          r.ident, app, r.intent, r.activityInfo, title, r.parent,
                          r.embeddedID, r.lastNonConfigurationInstances, config,
                          r.referrer, r.voiceInteractor);
}
```

这里调用方法时，已经在 H Callback 中将 ActivityClientRecord 替换为插件 Activity，而在 attach 的时候，也是将这个 token 作为参数写入进去。因而后续在 Client 段实际使用的是插件 Activity，尽管系统依然用着 StubActivity。

360 的 DP 方案，采用的也是类似的技术，大家可以最后进行下对比。

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年8月25日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
