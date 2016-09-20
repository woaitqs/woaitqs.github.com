---
layout: post
title: "Android 应用进程启动流程"
description: "分析了 Android 进程在启动时要完成的事情，理解其中发生的事情。"
keywords: "android ActivityManager 源码解析 binder activity"
category: "android"
tags: [android, program, binder, process]
---
{% include JB/setup %}

<div style="border:solid 1.5px #ccc;padding:20px 20px 10px 20px;margin-bottom: 20px;border-radius: 6px;">

          <span style="position: relative;padding: 5px 15px 5px 15px;background-color: white;bottom: 30px;font-size: 16px;font-weight: bolder;">摘要</span>

          <span style="position: relative;display: block;bottom: 15px;">
          本文是 Android 系统学习系列文章中的第一章节第二小节内容，从源码出发说明了 Android 应用进程是如何启动的，经过哪些进程的通力合作，它们是如何是设计的。对此系列感兴趣的同学，可以收藏这个链接 <a herf="http://www.woaitqs.cc/2016/06/16/android-system-learning.html">Android 系统学习</a>，也可以点击 <a href="http://www.woaitqs.cc/feed.xml">RSS订阅</a> 进行订阅。
          </span>
</div>

### 阅读的收益

讨论的内容也就是一个应用进程是如何启动的，私以为这一部分的内容颇为重要，即便不了解细节，也要知道其中的大体步骤。特别是针对我们应用开发者而言，理应了解我们的 App 是如何被启动的，App 中的组件是如何被系统服务调用和组织的。

讲应用进程启动的文章不是很多，也都没有说到点上，大抵都是对源码的堆叠，没有个人的理解在里面。如果非要看调用栈的话，在合适的地方挂上断点，或者通过输出异常栈的方式都可以看到，如下图所示。老罗的文章 [Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748) 事无巨细，但感觉还是没有说到点子上，因而这篇文章只做到抛砖引玉的作用，希望有更好的文章出现。细节是复杂的，原理是简单的，这里尽可能地从原理角度出发进行说明，加深大家的理解。

<!--break-->

对这篇文章阅读后，你能了解从用户点击 Launcher 上的 App 图标，到显示出 App 界面时主要发生的事情，而通过对这个一过程的了解，将知晓以下知识点。

- Android Process 的创建过程，以及 Activity Manager Service 是如何参与这个步骤，以及在其中扮演的角色？
- Android 中所谓的主线程是怎么回事？主线程是谁？又如何被创建的。
- Android 系统是如何节省进程创建开销的？

### 应用进程简介

在 Android 中每一个应用程序都被设计为单独的进程，应用程序也可以根据自己的需要去决定是否需要启用多个进程，不过总而言之都与其他应用程序和系统服务是相互独立的。从解耦和系统稳定性的角度上看都应该运行在不同的进程上，毕竟不能因为应用程序的崩溃就影响到其他应用进程或者系统服务进程。

应用进程不同于其他 Android 系统中的守护进程，当内存不够的时候，某些应用进程可能会被系统回收掉，因而应用进程也是有其生命周期的，更多信息参考 [Android Developer 官网对于 Process 的教程](https://developer.android.com/guide/components/processes-and-threads.html) 。Android 应用组件不一定要运行在单独的进程上，也可以运行在多个进程上，通过对 Android 组件指定运行的进程 `android:process`，即可让其运行在其他线程上。

每个应用进程都相当于一个 Sandbox 沙箱，Android 通过对每一个应用分配一个 UID，注意这里的 UID 不同于 Linux 系统的 User ID，可以将每个应用理解为一个 User
 ，只能对其目录下的内容具有访问和读写权限，这样就从根源上保护了其他应用程序，下图说明了其隔离效果。

 ![App Process Isolate](http://o8p68x17d.bkt.clouddn.com/Android-App-Processes-cropped.jpg)

----------------

### Zygote 进程

Zygote 的中文意思是受精卵，从这个意思里也可以看出 Zygote 进程是用来分裂复制(fork)的，实际上所有的 App 进程都是通过对 Zygote 进程的 Fork 得来的。当 [app_process](https://github.com/android/platform_frameworks_base/blob/master/cmds/app_process/app_main.cpp) 启动 Zygote 时，Zygote 会在其启动后，预加载必要的 Java Classes（相关列表查看 [预加载文件](https://github.com/android/platform_frameworks_base/blob/master/preloaded-classes)） 和 Resources，并启动 [System Server](http://www.woaitqs.cc/android/2016/05/26/android-binder-token.html) ，并打开 `/dev/socket/zygote` socket 去监听启动应用程序的请求。在下面的代码中，显示了 Zygote 进程如何启动，和加载 System Server 的。

![受精卵](http://worms.zoology.wisc.edu/dd2/echino/cleavage/files/page7_1.jpg)

```shell

service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
class main
socket zygote stream 660 root system
```

```java
public static void main(String argv[]) {
  // ...
  registerZygoteSocket(socketName); // 开启 Zygote socket.
  Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "ZygotePreload");
  EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
      SystemClock.uptimeMillis());
  preload(); // 预加载资源
  EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
      SystemClock.uptimeMillis());
  Trace.traceEnd(Trace.TRACE_TAG_DALVIK);
  // Finish profiling the zygote initialization.
  SamplingProfilerIntegration.writeZygoteSnapshot();
  // Do an initial gc to clean up after startup
  Trace.traceBegin(Trace.TRACE_TAG_DALVIK, "PostZygoteInitGC");
  gcAndFinalize(); // 触发 GC
  if (startSystemServer) { // 启动 System Server.
      startSystemServer(abiList, socketName);
  }
  // ...
}
```

![Fork Zygote 交互图](http://o8p68x17d.bkt.clouddn.com/zygote-fork-process.png)

那么为何要做这种设计呢？每个应用程序的运行，都需要依托于相应的运行环境，而这个就是 Davlik (ART) 虚拟机，但每次启动的开销较大，而通过对 Zygote 进程的 Fork，能够提升不小的效率。并且在这个工程中，采用了 Copy-on-Write 的方式，极大程度上地复用了 Zygote 上面的资源。更多信息也可以参考我这篇博文 [详解 Android 是如何启动的](http://www.woaitqs.cc/android/2016/06/15/how-android-launch-itself.html)。

---------------------

### App 应用进程启动

接下来的内容涉及到很多 Binder 通信相关的东西，因此在阅读本文前，建议查阅下 Binder 相关的文章，这里有下列文章供查考。

- [Android Binder 完全解析（一）概述](http://www.woaitqs.cc/android/2016/05/23/android-binder.html)
- [Android Binder 完全解析（二）设计详解](http://www.woaitqs.cc/android/2016/05/26/android-binder-token.html)
- [Android Binder 完全解析（三）AIDL实现原理分析](http://www.woaitqs.cc/android/2016/05/30/android-binder-proxy-and-token.html)

#### ActivityManager 架构

在我们编程过程中，涉及到许多 Activity 跳转的事情，在 Launcher 中点击 Icon 进行跳转也是同样的道理，调用 `context.startActivity(intent)` 方法。Launcher 出于一个进程，而启动的 App 则运行在另一个进程中，在这其中势必牵涉到跨进程 (IPC) 调用，这样复杂的过程显然需要一种中介者，或者一个系统来进行中转和管理，而这个服务就是 `ActivityManagerService`。

`ActivityManagerService` 作为一个守护进程运行在 Android Framework 中，如果让开发者直接接触这个类的话，就需要开发者自行处理 IPC 调用的问题，且这有不利于 Android 系统进行安全校验等工作，因而 Android 系统实现了 `ActivityManager`，通过这个 `ActivityManager` 作为一个入口，变相地和 `ActivityManagerService` 打交道。这种模式在 Android 系统中极为常见，类似的还有 `WifiManager`, `LocationManager`，`WindowsManager` 等等。而这些 Manger 在背后调用的东西就是前面提及的 Binder 机制。下面以 ActivityManger 为例看看其背后的运作方式。

Binder 体系架构可以分为 Client 和 Server 两端，为了更方便 Client 的调用，这次采用了 AIDL 的方式，具体参考链接 [Android Binder 完全解析（三）AIDL实现原理分析](http://www.woaitqs.cc/android/2016/05/30/android-binder-proxy-and-token.html) 。这里再用类比的方式来说明，方便大家理解。历史上有不少垂帘听政的故事，背后操作的人实际是通过控制傀儡来控制朝政，通过给傀儡皇帝传递命令，傀儡皇帝只是复述命令，起到传递的作用。更有甚者，不想去上朝的控权者，会通过手下的太监或者婢女，转述给傀儡皇上。这种模式被我们称为代理模式，`ActivityManger` 所使用的就是这种模式。

首先这里要针对要执行的命令进行抽象，这样掌权者、太监、皇上和朝政才能听懂。`IActivityManager` 就是对这个进行的抽象，点击查看 [源码](https://android.googlesource.com/platform/frameworks/base/+/c80f952/core/java/android/app/IActivityManager.java)，这几种就包括常见的 startActivity, showWaitingForDebugger, finishActivity 等等方法。`ActivityManagerProxy` 就相当于其他的太监或者婢女，Proxy 不需要懂具体的业务，只需要把指令传递过去就行。`ActivityManagerService` 就是具体的执行者，就是大臣们。`ActivityManger` 则就是具体的业务逻辑的外观类(参加GOF的设计模式)，也就是具体的掌权者们。它们的关系如下图所示：

![ActivityManager Service](http://o8p68x17d.bkt.clouddn.com/activity_manager_extend_relation.png)

#### 源码分析从 Launcher 到 ActivityManager 启动步骤

接下来分析下，在源码里是具体操作的，也验证我们前面的说法。

(1) 点击 Launcher 的图标，会调用到 Activity 的 startActivity 方法。在继续往下看过去，这个里面会调用到 startActivityForResult 方法，在 startActivityForResult 方法中，殊途同归，最后会调用 `mInstrumentation.execStartActivity` 方法。

```java
@Override
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}
```

```java
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
    if (mParent == null) {
        // ### ...
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
    } else {
        // ### ...
    }
}
```

(2) Instrumentation 执行 execStartActivity 方法。参数里面中的 contextThread 和 token 对象都是 IBinder 类型，而 Binder 可以在跨进程调用中依旧充当 Token 的角色，在多进程中由 Binder Driver 保证依然可以是唯一的。

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    // monitor ...
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess();
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
    }
    return null;
}
```

(3) ActivityManagerNative.getDefault().startActivity。getDefault 中实际返回的就是 Proxy 对象，在实际中只起到代理的重要，并不进行逻辑处理。

先看看 ActivityManagerNative.getDefault() 中的实现。

```java
/**
 * Retrieve the system's default/global activity manager.
 */
static public IActivityManager getDefault() {
    return gDefault.get();
}
```

IActivityManager 根据前文的描述即是对于可操作接口的抽象，Singleton 则是对单例对象的封装。也就是说 gDefault 返回了实现 IActivityManager 的单例。

```java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");
        if (false) {
            Log.v("ActivityManager", "default service binder = " + b);
        }
        IActivityManager am = asInterface(b);
        if (false) {
            Log.v("ActivityManager", "default service = " + am);
        }
        return am;
    }
};

/**
 * Cast a Binder object into an activity manager interface, generating
 * a proxy if needed.
 */
static public IActivityManager asInterface(IBinder obj) {
    if (obj == null) {
        return null;
    }
    IActivityManager in =
        (IActivityManager)obj.queryLocalInterface(descriptor);
    if (in != null) {
        return in;
    }

    return new ActivityManagerProxy(obj);
}
```

在 asInterface 中返回了 ActivityManagerProxy, 这就是前文提及的 太监和婢女 角色，我们再看看 ActivityManagerProxy 内部是如何工作的。

(4) ActivityManagerProxy 的实现。从源码里面可以看出，ActivityManagerProxy 将远程 Binder 作为构造函数的参数，而在 startActivity 方法中，通过远程 Binder 对象的 `transact` 方法，将参数写入到 `data` 中，在远程执行完毕后，结果写入到 `reply` 里。这里实实在在地起到了 Proxy 的作用，只负责数据的传输。重点在下面这行代码：

```java
mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
```

```java
class ActivityManagerProxy implements IActivityManager {

  public ActivityManagerProxy(IBinder remote) {
        mRemote = remote;
  }

  public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
              String resolvedType, IBinder resultTo, String resultWho, int requestCode,
              int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
          Parcel data = Parcel.obtain();
          Parcel reply = Parcel.obtain();
          data.writeInterfaceToken(IActivityManager.descriptor);
          data.writeStrongBinder(caller != null ? caller.asBinder() : null);
          data.writeString(callingPackage);
          intent.writeToParcel(data, 0);
          data.writeString(resolvedType);
          data.writeStrongBinder(resultTo);
          data.writeString(resultWho);
          data.writeInt(requestCode);
          data.writeInt(startFlags);
          if (profilerInfo != null) {
              data.writeInt(1);
              profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
          } else {
              data.writeInt(0);
          }
          if (options != null) {
              data.writeInt(1);
              options.writeToParcel(data, 0);
          } else {
              data.writeInt(0);
          }
          mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
          reply.readException();
          int result = reply.readInt();
          reply.recycle();
          data.recycle();
          return result;
    }

    // other methods.

}
```

(5) ActivityManagerService 的调用。

在上部分提及的 ActivityManagerProxy 中在构造函数里传入的 mRemote 远程Binder 是什么了？答案就在前面提及的 gDefault 里面。

```java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");
        if (false) {
            Log.v("ActivityManager", "default service binder = " + b);
        }
        IActivityManager am = asInterface(b);
        if (false) {
            Log.v("ActivityManager", "default service = " + am);
        }
        return am;
    }
};
```

```java
IBinder b = ServiceManager.getService("activity");
```

上面这段代码返回的即是 `ActivityManagerService`。所有的系统服务都是 IBinder 对象，即他们必须支持远程调用。而每个系统服务都会通过在 ServiceManager 注册别名的方式，告知 ServiceManager 通过相应的别名即可访问到我。而 activity 正是 ActivityManagerService 的别名。

#### 从 ActivityManagerService 到 进程启动

ActivityManagerService 在接受到相应的 Intent 请求后（Activity、Broadcast、Service、ContentProvider），会查看是否需要进行新建进程的工作，这里以 Activity 为例，其他组件的步骤与此原理相同，就不再赘述。

(1) ActivityManagerService 在启动 Activity 之前，首先通过 resolveIntent 方法，来得到相应的 ResolveInfo，其后通过调用 startActivityLocked 往下启动 Activity。

```java
try {
    ResolveInfo rInfo =
        AppGlobals.getPackageManager().resolveIntent(
                intent, null,
                PackageManager.MATCH_DEFAULT_ONLY
                | ActivityManagerService.STOCK_PM_FLAGS, userId);
    aInfo = rInfo != null ? rInfo.activityInfo : null;
    aInfo = mService.getActivityInfoForUser(aInfo, userId);
} catch (RemoteException e) {
    aInfo = null;
}

int res = startActivityLocked(caller, intent, resolvedType, aInfo,
        voiceSession, voiceInteractor, resultTo, resultWho,
        requestCode, callingPid, callingUid, callingPackage,
        realCallingPid, realCallingUid, startFlags, options,
        componentSpecified, null, container, inTask);
```

(2) startSpecificActivityLocked 方法，判断是否需要新建进程。从代码中看出，这里对 ProcessRecord 进行了判断，ProcessRecord 就是响应的进程记录，如果存在相应的进程，就启动相应的 Activity, 否则将创建进程。`mService.startProcessLocked` 这个方法实现了开启进程，下面再看看里面的实现。

```java
void startSpecificActivityLocked(ActivityRecord r,
        boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
            r.info.applicationInfo.uid, true);

    r.task.stack.setLaunchTime(r);

    if (app != null && app.thread != null) {
        try {
            // ignore some code...
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
    }

    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
            "activity", r.intent.getComponent(), false, false, true);
}
```

(3) startProcessLocked 在内部调用了 `Process.start` 方法，并且指定了 `android.app.ActivityThread` 作为进程的入口，进程启动后，将调用 `android.app.ActivityThread` 的 main 方法。

```java
private final void startProcessLocked(ProcessRecord app, String hostingType,
        String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
        // ...
        // Start the process.  It will either succeed and return a result containing
        // the PID of the new process, or else throw a RuntimeException.
        boolean isActivityProcess = (entryPoint == null);
        if (entryPoint == null) entryPoint = "android.app.ActivityThread";
        checkTime(startTime, "startProcess: asking zygote to start proc");
        Process.ProcessStartResult startResult = Process.start(entryPoint,
        app.processName, uid, uid, gids, debugFlags, mountExternal,
        app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
        app.info.dataDir, entryPointArgs);
        // ...
}
```

(4) 在 Process.start 方法中，实际调用的是 `startViaZygote` 方法，在这个方法里通过 openZygoteSocketIfNeeded 打开 Zygote 的 socket，并通过 `zygoteSendArgsAndGetResult` 进行交互。

根据前文提及的内容，zygote 开启了一个 socket 监听功能，监听需要创建 Process 的请求，因而在这里，我们去查看 Zygote 的代码，看其是怎么监听请求的。

```java
return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
```

(5) ZygoteInit.runSelectLoop 方法，从源码的实现中可以看出，这里是不断地取 Socket 建立的链接(ZygoteConnection)，然后调用 ZygoteConnection 中的 runOnce 方法。

```java
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
    ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
    FileDescriptor[] fdArray = new FileDescriptor[4];

    fds.add(sServerSocket.getFileDescriptor());
    peers.add(null);

    int loopCount = GC_LOOP_COUNT;
    while (true) {
        int index;

        /*
         * Call gc() before we block in select().
         * It's work that has to be done anyway, and it's better
         * to avoid making every child do it.  It will also
         * madvise() any free memory as a side-effect.
         *
         * Don't call it every time, because walking the entire
         * heap is a lot of overhead to free a few hundred bytes.
         */
        if (loopCount <= 0) {
            gc();
            loopCount = GC_LOOP_COUNT;
        } else {
            loopCount--;
        }


        try {
            fdArray = fds.toArray(fdArray);
            index = selectReadable(fdArray);
        } catch (IOException ex) {
            throw new RuntimeException("Error in select()", ex);
        }

        if (index < 0) {
            throw new RuntimeException("Error in select()");
        } else if (index == 0) {
            ZygoteConnection newPeer = acceptCommandPeer(abiList);
            peers.add(newPeer);
            fds.add(newPeer.getFileDescriptor());
        } else {
            boolean done;
            done = peers.get(index).runOnce();

            if (done) {
                peers.remove(index);
                fds.remove(index);
            }
        }
    }
}
```

(6) ZygoteConnection.runOnce 方法里，fork 了 Zygote 进程，这就是 `应用进程` 了，并返回相应的 process id，其具体实现是本地方法，这里就不再深究了。如果得到相应的 Pid，接下来看看应用进程是如何初始化的。

```java
pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
        parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
        parsedArgs.niceName, fdsToClose, parsedArgs.instructionSet,
        parsedArgs.appDataDir);

try {
    if (pid == 0) {
        // in child
        IoUtils.closeQuietly(serverPipeFd);
        serverPipeFd = null;
        handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);

        // should never get here, the child is expected to either
        // throw ZygoteInit.MethodAndArgsCaller or exec().
        return true;
    } else {
        // in parent...pid of < 0 means failure
        IoUtils.closeQuietly(childPipeFd);
        childPipeFd = null;
        return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs);
    }
} finally {
    IoUtils.closeQuietly(childPipeFd);
    IoUtils.closeQuietly(serverPipeFd);
}
```

(7) handleChildProc 方法，其中的重点就在 RuntimeInit.zygoteInit 方法 和 ZygoteInit.invokeStaticMain 方法。一个调用初始化了相应的进程，另一个在调用了进程的 main 方法。

```java
private void handleChildProc(Arguments parsedArgs,
        FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)
        throws ZygoteInit.MethodAndArgsCaller {

    if (parsedArgs.runtimeInit) {
        if (parsedArgs.invokeWith != null) {
            WrapperInit.execApplication(parsedArgs.invokeWith,
                    parsedArgs.niceName, parsedArgs.targetSdkVersion,
                    pipeFd, parsedArgs.remainingArgs);
        } else {
            RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                    parsedArgs.remainingArgs, null /* classLoader */);
        }
    }

    try {
        ZygoteInit.invokeStaticMain(cloader, className, mainArgs);
    } catch (RuntimeException ex) {
        logAndPrintError(newStderr, "Error starting.", ex);
    }

}
```

在 RuntimeInit.zygoteInit 方法里实现了相应的 AndroidRuntime 初始化，并初始化 Binder Driver 相关的文件，同时设置了 UncaughtHandler（应用程序崩溃）。在这个方法调用结束后，应用进程就具备与相应系统服务进行 IPC 通信的能力。

(8) ZygoteInit.invokeStaticMain 通过反射调用了 ZygoteInit.MethodAndArgsCaller，而这个就是相应的 `ActivityThread.java` 中的 main 方法，其所运行的进程，就是大家耳熟能祥的主线程，也成为 UI 线程。

```java
static void invokeStaticMain(ClassLoader loader,
        String className, String[] argv)
        throws ZygoteInit.MethodAndArgsCaller {
    Class<?> cl;

    try {
        cl = loader.loadClass(className);
    } catch (ClassNotFoundException ex) {
        throw new RuntimeException(
                "Missing class when invoking static main " + className,
                ex);
    }

    /*
     * This throw gets caught in ZygoteInit.main(), which responds
     * by invoking the exception's run() method. This arrangement
     * clears up all the stack frames that were required in setting
     * up the process.
     */
    throw new ZygoteInit.MethodAndArgsCaller(m, argv);
}
```

到此为止，整个应用进程启动完毕。

![get process](http://img.blog.csdn.net/20131209165210656?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveWFuZ3dlbjEyMw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

----------------------------

### 应用进程启动总结

![Android Process 启动流程总结](http://o8p68x17d.bkt.clouddn.com/app_launch_summary.jpg)

Launcher 中的 Icon 点击，broadcast 发送，启动 Service 等组件见的跳转，都会通过 `AndroidManagerProxy` 来进行中转，而 `AndroidManagerProxy` 通过向 `SystemServer` 请求名为 `Activity` 的 `ActivityManagerService` 的 Binder 对象，这个 Binder 对象可以粗略地看作是 `ActivityManagerService` 的句柄，从 Binder 对象可实际操作 `ActivityManagerService`。

`ActivityManagerService` 在实际启动相应组件时，会先判断是否有相应的 ProcessRecord，如果不存在，就需要新建进程，这个进程就是相应的应用进程。`ActivityManagerService` 通过 Socket 通信的方式和 Zygote 进行协商，Zygote 在其监听的 `/dev/socket/zygote` socket 中发现有需要创建进程的请求后，会 fork 自身，并返回相应的 Process Id。这个 Process 会进行相应的初始化，使得其具备与系统服务进行 IPC 通信的能力，在此之后，调用 `ActivityThread` 中的 main 方法，开启 Looper，主线程启动。到此为止，整个应用进程启动完毕。

----------------------------

### 参考文献

1. [Android应用程序进程启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6747696)
2. [Android应用程序线程消息循环模型分析](http://blog.csdn.net/luoshengyang/article/details/6905587)
3. [startActivity流程分析(一)](http://gityuan.com/2016/03/12/start-activity/)
4. [Android四大组件 Activity启动过程源码分析](http://mouxuejie.com/blog/2016-03-12/activity-launch-analysis/)
5. [Android Application Launch](http://multi-core-dump.blogspot.hk/2010/04/android-application-launch.html)

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年6月16日
* 社交媒体：[weibo](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
