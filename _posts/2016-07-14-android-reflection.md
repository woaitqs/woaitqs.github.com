---
layout: post
title: "Android 极简反射教程，及应用示例"
description: "介绍了 Android 反射相关的方法"
category: "android"
keywords: "android, 反射, 教程"
tags: [android, reflection, program, plugin]
---
{% include JB/setup %}

<div style="border:solid 1.5px #ccc;padding:20px 20px 10px 20px;margin-bottom: 20px;border-radius: 6px;">

          <span style="position: relative;padding: 5px 15px 5px 15px;background-color: white;bottom: 30px;font-size: 16px;font-weight: bolder;">摘要</span>

          <span style="position: relative;display: block;bottom: 15px;">
          本文介绍了 Java 反射的基础知识，以及用一个实际的例子介绍了如何在 Android 开发中应用反射。
          </span>
</div>

<!--break-->

### Java 反射简介

Java 程序的运行需要相应的环境（Java Runtime Environment）, 而这其中最有名的就是 JVM，JVM 提供了动态运行字节码的能力，除了 JVM 帮我们做链接、加载字节码相关的事情外，也通过反射提供给我们动态修改的能力。反射使得我们能够在运行时，不知道类名、方法名的情况下查看其中的接口、方法和字段等信息，另一方面，也可以动态调用方法、新建对象，甚至篡改字段值。

那介绍了反射是干嘛之后，反射能在实际的工作中发挥什么作用吗？Android 系统在设计的时候，出于安全和架构的考虑，利用了 Java 权限相关的东西（private，package等等，以及 @hide 这个注解）使得我们无法访问某些字段，或者方法。但在实际开发过程中，这些隐藏的字段或者方法却能提供给我们非常想要的特性。在这种矛盾的情况下，反射就能满足我们的需求，像是打开隐藏关卡的一把钥匙。

总结起来就是，反射提供了一种与 Class 文件进行动态交互的机制。例如在下面的入口函数中，就可以看到 HashMapClass 里所有的方法。

```java
public class HashMapClass extends HashMap {

  public static void main(String[] args) {
    Method[] methods = HashMapClass.class.getMethods();

    for (Method method : methods) {
      System.out.println("method name is " + method.getName());
    }
  }

}
```

### Class 类简介

在进行接下来的反射教程中，首先应该了解 Class Object。Java 中所有的类型，包括 int、float 等基本类型，都有与之相关的 Class 对象。如果知道对应的 Class name，可以通过 `Class.forName()` 来构造相应的 Class 对象，如果没有对应的 class，或者没有加载进来，那么会抛出 ClassNotFoundException 对象。

Class 封装了一个类所包含的信息，主要的接口如下:

```java
    try {
      Class mClass = Class.forName("java.lang.Object");

      // 不包含包名前缀的名字
      String simpleName = mClass.getSimpleName();

      // 类型修饰符, private, protect, static etc.
      int modifiers = mClass.getModifiers();
      // Modifier 提供的一些用于判读类型的静态方法.
      Modifier.isPrivate(modifiers);

      // 父类的信息
      Class superclass = mClass.getSuperclass();

      // 构造函数
      Constructor[] constructors = mClass.getConstructors();

      // 字段类型
      Field[] fields = mClass.getFields();
    } catch (ClassNotFoundException e) {
      e.printStackTrace();
    }
```

### 常用反射方法

下面列举一些反射常见的应用场景，主要从 Student 这个类进行入手。

```java
public class Student {

  private final String name;
private int grade = 1;

  public Student(String name) {
    this.name = name;
  }

  public String getName() {
    return name;
  }

  private int getGrade() {
    return grade;
  }

  private void goToSchool() {
    System.out.println(name + " go to school!");
  }
}
```

#### 1）反射构建 Student 对象

```java
try {
  Class studentClass = Student.class;

  // 参数类型为一个 String 的构造函数
  Constructor constructor = studentClass.getConstructor(new Class[]{String.class});

  // 实例化 student 对象
  Student student = (Student)constructor.newInstance("Li Lei");
  System.out.print(student.getName());
} catch (ReflectiveOperationException e) {
  e.printStackTrace();
}
```

#### 2）反射修改私有变量

```java
try {
  Student student = new Student("Han MeiMei");
  System.out.println("origin grade is " + student.getGrade());

  Class studentClass = Student.class;
  // 获取声明的 grade 字段，这里要注意 getField 和 getDeclaredField 的区别
  Field gradeField = studentClass.getDeclaredField("grade");

  // 如果是 private 或者 package 权限的，一定要赋予其访问权限
  gradeField.setAccessible(true);

  // 修改 student 对象中的 Grade 字段值
  gradeField.set(student, 2);
  System.out.println("after reflection grade is " + student.getGrade());

} catch (ReflectiveOperationException e) {
  e.printStackTrace();
}
```

#### 3）反射调用私有方法

```java
try {
  Student student = new Student("Han MeiMei");

  // 获取私有方法，同样注意 getMethod 和 getDeclaredMethod 的区别
  Method goMethod = Student.class.getDeclaredMethod("goToSchool", null);
  // 赋予访问权限
  goMethod.setAccessible(true);

  // 调用 goToSchool 方法。
  goMethod.invoke(student, null);

} catch (ReflectiveOperationException e) {
  e.printStackTrace();
}
```

### Android 反射应用示例

学以致用，现在我们来通过实际的例子，来看看如何利用 Java 的反射特性来完成一些牛逼的功能。设想我们想通过插件的方式，来启动一个未注册的 Activity，这就会涉及到很多问题，其中之一就是如何赋予这些插件 Activity 生命周期。这个例子就是通过反射的方式，来手动地进行 Activity 生命周期的通知。

#### 1）了解并熟悉代码细节

要实现上述的功能，第一步就是要知道 Activity 的生命周期是如何运作的，要对代码细节有所了解。因为反射所操作的对象是具体的 Class 对象，如果不清楚源码细节，反射将无从说起。

篇幅所限，具体的原理又较为复杂，这里列出链接 [Android 插件化原理解析——Activity生命周期管理](http://weishu.me/2016/03/21/understand-plugin-framework-activity-management/), 有兴趣的同学可以自行查看，在这只进行大体上的说明。

Activity 生命周期与 `ActivityThread` 息息相关，我们来进行各个突破，先看看 Activity 是怎么启动的。当需要启动 Activity 时，ActivityManagerService 会通过 Binder 机制向 ActivityThread 发送消息，经过链式地调用后，会执行到`scheduleLaunchActivity` 这个方法，我们看看其内部的实现。

```java
// we use token to identify this activity without having to send the
// activity itself back to the activity manager. (matters more with ipc)
@Override
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
        ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
        CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
        int procState, Bundle state, PersistableBundle persistentState,
        List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
        boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

    updateProcessState(procState, false);

    ActivityClientRecord r = new ActivityClientRecord();

    r.token = token;
    r.ident = ident;
    r.intent = intent;
    r.referrer = referrer;
    r.voiceInteractor = voiceInteractor;
    r.activityInfo = info;
    r.compatInfo = compatInfo;
    r.state = state;
    r.persistentState = persistentState;

    r.pendingResults = pendingResults;
    r.pendingIntents = pendingNewIntents;

    r.startsNotResumed = notResumed;
    r.isForward = isForward;

    r.profilerInfo = profilerInfo;

    r.overrideConfig = overrideConfig;
    updatePendingConfiguration(curConfig);

    sendMessage(H.LAUNCH_ACTIVITY, r);
}
```

注意最后的 `sendMessage` 方法，说明内部是采用的 handler 机制来进行通信的。在这篇文章中 [Android 应用进程启动流程](http://www.woaitqs.cc/android/2016/06/21/activity-service.html) 提及到当应用进程启动后，会调用 `ActivityThread` 的 main 方法，并在这个方法中进行相应的消息循环初始化，其后在主线程上的消息传递都是通过 `ActivityThread` 中的 `H` 这个内部来进行初始化，这里的 `H` 就是可能的突破口之一。

```java
private class H extends Handler {
    public static final int LAUNCH_ACTIVITY         = 100;
    public static final int PAUSE_ACTIVITY          = 101;
    public static final int PAUSE_ACTIVITY_FINISHING= 102;
    public static final int STOP_ACTIVITY_SHOW      = 103;
    public static final int STOP_ACTIVITY_HIDE      = 104;
    public static final int SHOW_WINDOW             = 105;
    public static final int HIDE_WINDOW             = 106;
    public static final int RESUME_ACTIVITY         = 107;
    public static final int SEND_RESULT             = 108;
    public static final int DESTROY_ACTIVITY        = 109;
    public static final int BIND_APPLICATION        = 110;
    public static final int EXIT_APPLICATION        = 111;
    public static final int NEW_INTENT              = 112;
    public static final int RECEIVER                = 113;
    public static final int CREATE_SERVICE          = 114;
    public static final int SERVICE_ARGS            = 115;
    public static final int STOP_SERVICE            = 116;

    //......

    public void handleMessage(Message msg){
    	// ...
    }
}
```

既然发现通信是通过 `H` 这个 Handler 来完成的，那么再看看 Handler 的实现原理，这里也有一篇文章供参考, [Android Handler机制全解析](http://www.woaitqs.cc/android/2016/06/06/android-handler.html) 。Handler 在内部维护着一个 callback 对象，当有消息发生时，会通过这个 callback 往外发送消息。

```java
/**
 * Handle system messages here.
 */
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

如果能够替换 callback 变量，这样消息就可以传递到替换后的 callback 里了。这样是否能达到我们的目的？

#### 2) 验证是否满足反射条件

如果我们反射的对象，拥有多个实例，那么我们就需要在不同的地方进行处理，显然这样会额外增加实现的复杂度，因而反射尽量在 `单例`或者`静态` 实例上完成，代码的复杂度能提升不少。

在前面提到了通过替换 callback 的方式，这样是否可行，我们来验证下。首先 `H` 是放置在 ActivityThread 这个类里面的，而 ActivityThread 运行的线程就是主线程，我们知道每一个应用都拥有一个主线程，因而这里的 ActivityThread 只存在一份，进而也可以保证 `H` 的唯一性。

另一方面，ActivityThread 在内部也维护了 currentActivityThread 这个变量，虽然由于 API 的访问限制，不能直接访问，但也同样可以由反射拿到。

至此，可以证明这种方式理论上是可以成功的。

#### 3）实施代码细节，并在合适的地方进行代码注入

首先，我们自定义出自定义的 callback 对象，这个 callback 作为 `H` 中 callback 的代理，这里需要注意的是 msg 的定义要和底层保持一致，代码如下：

```java
public class LaunchCallback implements Handler.Callback {

  public static final int LAUNCH_ACTIVITY = 100;

  private Handler.Callback originCallback;

  public LaunchCallback(Handler.Callback originCallback) {
    this.originCallback = originCallback;
  }

  @Override
  public boolean handleMessage(Message msg) {

    if (msg.what == LAUNCH_ACTIVITY) {
      Toast.makeText(
          VApp.getApp().getApplicationContext(),
          "activity is going to launch! ", Toast.LENGTH_SHORT).show();
    }

    return originCallback.handleMessage(msg);
  }

}
```

通过前文提及的反射方法，将运行的 callback 替换为自定义的 callback，代码如下：

```java
public class InjectTool {

  public static void dexInject() {
    try {

      // 通过反射调用 ActivityThread 的静态方法, 获取 currentActivityThread
      Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
      Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
      currentActivityThreadMethod.setAccessible(true);
      Object currentActivityThread = currentActivityThreadMethod.invoke(null);

      // 获取 currentActivityThread 这个示例中的 mH
      Field handlerField = activityThreadClass.getDeclaredField("mH");
      handlerField.setAccessible(true);
      Handler handler = (Handler) handlerField.get(currentActivityThread);

      // 修改 mH 中的 callback 字段
      Field callbackField = Handler.class.getDeclaredField("mCallback");
      callbackField.setAccessible(true);
      Handler.Callback callback = (Handler.Callback) callbackField.get(handler);

      callbackField.set(handler, new LaunchCallback(callback));
    } catch (IllegalArgumentException | NoSuchMethodException | IllegalAccessException
        | InvocationTargetException | ClassNotFoundException | NoSuchFieldException e) {
      e.printStackTrace();
    }
  }

}
```

在 Application 中进行注入，完成修改的落地

```java
@Override
protected void attachBaseContext(Context base) {
    super.attachBaseContext(base);
    InjectTool.dexInject();
}
```

实际运行的效果，如图所示：

![activity launch](http://o8p68x17d.bkt.clouddn.com/monitor_activity_launch.png)

activity 其他的生命周期也可以同样处理，这里就不再赘述了。

#### 4）反射使用总结

在通过反射实现相关功能的时候，第一件事情就是认真地阅读源码，理清其中的脉络，其后找寻其中的突破点，这些点一般为 static 方法或者单例对象，最后才是代码落地。

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年7月14日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
