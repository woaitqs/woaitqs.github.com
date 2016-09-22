---
layout: post
title: "从源码出发深入理解 Android Service"
description: "从源码出发深入理解 Android Service"
keywords: "android source service"
category: "android"
tags: [android, service]
---
{% include JB/setup %}

<div style="border:solid 1.5px #ccc;padding:20px 20px 10px 20px;margin-bottom: 20px;border-radius: 6px;">

          <span style="position: relative;padding: 5px 15px 5px 15px;background-color: white;bottom: 30px;font-size: 16px;font-weight: bolder;">摘要</span>

          <span style="position: relative;display: block;bottom: 15px;">
          本文是 Android 系统学习系列文章中的第三章节的内容，介绍了 Android Service 相关的基础知识，然后从源码的角度上分析 Service 的一些实现原理。对此系列感兴趣的同学，可以收藏这个链接 <a herf="http://www.woaitqs.cc/2016/06/16/android-system-learning.html">Android 系统学习</a>，也可以点击 <a href="http://www.woaitqs.cc/feed.xml">RSS订阅</a> 进行订阅。
          </span>
</div>

<!--break-->

------------------------------------------

### 0X00 Service 基础知识

Service 作为 Android 提供的四大组件之一，主要负责一些没有前台显示的后台任务。即使应用本身不再可见，Service 的属性也能使得其在后台运行。除此之外，Service 也可以通过 Binder 机制，与界面甚至其他应用进行进程间通信，以实现相应的交互。这里需要简单说明的是，既然是后台任务，为什么不选用 Thread 了？选用 Service 和 Thread 的主要区别在于需不需要在应用不可见的时候依然保留。举例来说，新闻详情页面的数据请求，只用在当前页面生效，而音乐播放这些后台任务就可以通过 Service 的方式来实现。

关于如何使用 Service，官方教程已经说明得足够详细了，如果对这些用法，还有不清晰的地方，请戳这里进行查看，-> [官方教程](https://developer.android.com/guide/components/services.html)。官方教程里面包括，startService 和 bindService 的区别，在不同场景下应该选用哪种 Service 实现方式。

![Service 生命周期](http://o8p68x17d.bkt.clouddn.com/service_lifecycle.png)

------------------------

### 0X01 startService 调用流程

从前面的教程里面，可以知道 Service 的启动一般有两种方式，分别是 bindService 和 startService。这里主要说明 startService， 具体的实现逻辑在 ContextImpl 中，我们看看源码是怎么实现的。

```java
@Override
public ComponentName startService(Intent service) {
    warnIfCallingFromSystemProcess();
    return startServiceCommon(service, mUser);
}
```

接下来，看看方法内部具体是怎么实现的。

```java
private ComponentName startServiceCommon(Intent service, UserHandle user) {
    try {
        validateServiceIntent(service);
        service.prepareToLeaveProcess();
        ComponentName cn = ActivityManagerNative.getDefault().startService(
            mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                        getContentResolver()), getOpPackageName(), user.getIdentifier());
        // ignore some codes...
        return cn;
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
}
```

从上面的代码可以看到，这里是通过 ActivityManagerNative 来执行的。如果看过我的另一篇文章，[Android Activity 生命周期是如何实现的](http://www.woaitqs.cc/android/2016/07/19/how-activity-lifecircle-work), 可能会觉得很熟悉。事实上这里采用的机制就是同样的。

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

ActivityManagerNative 的 getDefault 方法是这么实现的。可以看到，gDefault 是类型为 IActivityManager 的 Binder 对象。而这个 Binder 对象可以看做是在 System Server 中的 ActivityManagerService 的句柄，通过这个 Binder 对象，可以跨进程调用 ActivityManagerService。

如果上述内容不容易理解的话，我们可以类比地来看这个问题。我们遥控电视的时候，例如进行增加音量的操作，这个操作实际不是由遥控器完成的，而是电视中的电子元件完成的。遥控器会和电视进行协商，先声明有哪些操作可以执行，然后将这些协商后的操作在遥控器端和电视端 📺 都实现，区别在于电视机是真的实现，而遥控器只是发送操作指令而已。前面代码中提及的 gDefault 就是 ActivityManagerService 的遥控器。

![电视遥控器](http://ec1img.pchome.com.tw/pic/v1/data/item/201011/A/F/A/A/1/Z/AFAA1Z-A50974180000_4cf4d79632dd5)

接着往下看看电视端是具体怎么操作的，这里的电视端就是 ActivityManagerService.

```java
@Override
public ComponentName startService(IApplicationThread caller, Intent service,
        String resolvedType, String callingPackage, int userId)
        throws TransactionTooLargeException {
    enforceNotIsolatedCaller("startService");

    // ignore some codes...

    synchronized(this) {
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();
        ComponentName res = mServices.startServiceLocked(caller, service,
                resolvedType, callingPid, callingUid, callingPackage, userId);
        Binder.restoreCallingIdentity(origId);
        return res;
    }
}
```

看起来具体的逻辑，都在类型为 ActiveServices 的 mServices 对象中。ActiveServices 是 AMS 专门用来管理 Service 的类，大部分和 Services 相关的逻辑都在这里面。

```java
ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
        int callingPid, int callingUid, String callingPackage, int userId)
        throws TransactionTooLargeException {
    if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "startService: " + service
            + " type=" + resolvedType + " args=" + service.getExtras());

    // 调用进程的优先级是否足够
    final boolean callerFg;
    if (caller != null) {
        // 能否找到相应的 Process
        final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
        if (callerApp == null) {
            throw new SecurityException(
                    "Unable to find app for caller " + caller
                    + " (pid=" + Binder.getCallingPid()
                    + ") when starting service " + service);
        }
        callerFg = callerApp.setSchedGroup != Process.THREAD_GROUP_BG_NONINTERACTIVE;
    } else {
        callerFg = true;
    }

    // ignore some codes.

    final ServiceMap smap = getServiceMap(r.userId);
    boolean addToStarting = false;
    if (!callerFg && r.app == null && mAm.mStartedUsers.get(r.userId) != null) {
        ProcessRecord proc = mAm.getProcessRecordLocked(r.processName, r.appInfo.uid, false);
        if (proc == null || proc.curProcState > ActivityManager.PROCESS_STATE_RECEIVER) {
            // 避免同时有多个服务启动时，造成的性能问题，如果超过阈值后，就进行延迟
            if (r.delayed) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "Continuing to delay: " + r);
                return r.name;
            }
            if (smap.mStartingBackground.size() >= mMaxStartingBackground) {
                // Something else is starting, delay!
                Slog.i(TAG_SERVICE, "Delaying start of: " + r);
                smap.mDelayedStartList.add(r);
                r.delayed = true;
                return r.name;
            }
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "Not delaying: " + r);
            addToStarting = true;
        } else if (proc.curProcState >= ActivityManager.PROCESS_STATE_SERVICE) {
            addToStarting = true;
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                    "Not delaying, but counting as bg: " + r);
        }
    }

    return startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
}
```

可以看到 startServiceLocked 主要进行了权限校验和为性能进行的调度，具体的逻辑，还在 startServiceInnerLocked 方法里面。

```java
ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
        boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {
    ProcessStats.ServiceState stracker = r.getTracker();
    if (stracker != null) {
        // 跟踪 Service 内存试用状况
        stracker.setStarted(true, mAm.mProcessStats.getMemFactorLocked(), r.lastActivity);
    }
    r.callStart = false;
    synchronized (r.stats.getBatteryStats()) {
        // 跟踪电池占用情况
        r.stats.startRunningLocked();
    }
    // 实际启动 Service
    String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false);
    if (error != null) {
        return new ComponentName("!!", error);
    }

    // 设置超时时间
    if (r.startRequested && addToStarting) {
        boolean first = smap.mStartingBackground.size() == 0;
        smap.mStartingBackground.add(r);
        r.startingBgTimeout = SystemClock.uptimeMillis() + BG_START_TIMEOUT;
        if (DEBUG_DELAYED_SERVICE) {
            RuntimeException here = new RuntimeException("here");
            here.fillInStackTrace();
            Slog.v(TAG_SERVICE, "Starting background (first=" + first + "): " + r, here);
        } else if (DEBUG_DELAYED_STARTS) {
            Slog.v(TAG_SERVICE, "Starting background (first=" + first + "): " + r);
        }
        if (first) {
            smap.rescheduleDelayedStarts();
        }
    } else if (callerFg) {
        smap.ensureNotStartingBackground(r);
    }

    return r.name;
}
```

接下来看看 bringUpServiceLocked 是如何实现的，这里就是实际启动 Service 的地方。

```java
private final String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
        boolean whileRestarting) throws TransactionTooLargeException {

    if (r.app != null && r.app.thread != null) {
        // 目标Service已经启动
        sendServiceArgsLocked(r, execInFg, false);
        return null;
    }

    if (!whileRestarting && r.restartDelay > 0) {
        // 正在重启，或者等待重启，直接返回
        return null;
    }

    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bringing up " + r + " " + r.intent);

    // 从重启 Service 列表中移除
    if (mRestartingServices.remove(r)) {
        // 重试计数
        r.resetRestartCounter();
        clearRestartingIfNeededLocked(r);
    }

    // 移除 delayed 的标示
    if (r.delayed) {
        if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (bring up): " + r);
        getServiceMap(r.userId).mDelayedStartList.remove(r);
        r.delayed = false;
    }

    // Service 即将被启动，它所属的Package不能被停止
    try {
        AppGlobals.getPackageManager().setPackageStoppedState(
                r.packageName, false, r.userId);
    } catch (RemoteException e) {
    } catch (IllegalArgumentException e) {
        Slog.w(TAG, "Failed trying to unstop package "
                + r.packageName + ": " + e);
    }

    // isolated 变量主要是用于如下问题
    // Service 可以通过 AndroidManifest 中的指令指定在单独的进程上运行
    // 启动这个单独的进程，是一个耗时的过程，因而 isolated 标记起来后，
    // 可以避免在这个过程中又创建了一个进程
    final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
    final String procName = r.processName;
    ProcessRecord app;

    if (!isolated) {
        app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
        if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                    + " app=" + app);
        // 如果应用进程已经创建，而且 Looper 线程已经准备完毕
        // 那么就调用 realStartServiceLocked 实际启动 Service
        if (app != null && app.thread != null) {
            try {
                app.addPackage(r.appInfo.packageName, r.appInfo.versionCode, mAm.mProcessStats);
                realStartServiceLocked(r, app, execInFg);
                return null;
            } catch (TransactionTooLargeException e) {
                throw e;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting service " + r.shortName, e);
            }

        }
    } else {
        app = r.isolatedProc;
    }

    // 应用进程还未创建，这里需要先启动应用进程
    if (app == null) {
        if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                "service", r.name, false, isolated, false)) == null) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": process is bad";
            Slog.w(TAG, msg);
            bringDownServiceLocked(r);
            return msg;
        }
        if (isolated) {
            r.isolatedProc = app;
        }
    }


    if (!mPendingServices.contains(r)) {
        mPendingServices.add(r);
    }

    if (r.delayedStop) {
        // 准备停止 Service
        r.delayedStop = false;
        if (r.startRequested) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                    "Applying delayed stop (in bring up): " + r);
            stopServiceLocked(r);
        }
    }

    return null;
}
```

上面的代码较为复杂，这里进行下初步的总结。首先判断服务是否已经启动，或者正在重启中，则直接返回。其后，当前 Service 进程正在启动中，也直接返回。最后判断应用进程是否启动，如果没有启动进程，则先启动进程。

```java
private final void realStartServiceLocked(ServiceRecord r,
        ProcessRecord app, boolean execInFg) throws RemoteException {
    // 如果主进程不存在，抛出异常
    if (app.thread == null) {
        throw new RemoteException();
    }

    r.app = app;
    // 记录重启和最后活动时间
    r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

    final boolean newService = app.services.add(r);
    bumpServiceExecutingLocked(r, execInFg, "create");
    mAm.updateLruProcessLocked(app, false, null);
    mAm.updateOomAdjLocked();

    boolean created = false;
    try {
        if (LOG_SERVICE_START_STOP) {
            String nameTerm;
            int lastPeriod = r.shortName.lastIndexOf('.');
            nameTerm = lastPeriod >= 0 ? r.shortName.substring(lastPeriod) : r.shortName;
            EventLogTags.writeAmCreateService(
                    r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
        }
        synchronized (r.stats.getBatteryStats()) {
            r.stats.startLaunchedLocked();
        }
        // 确保 Package Opt 工作完成
        mAm.ensurePackageDexOpt(r.serviceInfo.packageName);
        app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
        // 通过 app.thread 的 binder 对象，通过创建目标 Service.
        app.thread.scheduleCreateService(r, r.serviceInfo,
                mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                app.repProcState);
        r.postNotification();
        created = true;
    } catch (DeadObjectException e) {
        Slog.w(TAG, "Application dead when creating service " + r);
        mAm.appDiedLocked(app);
        throw e;
    } finally {
        if (!created) {
            // Keep the executeNesting count accurate.
            final boolean inDestroying = mDestroyingServices.contains(r);
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);

            // Cleanup.
            if (newService) {
                app.services.remove(r);
                r.app = null;
            }

            // Retry.
            if (!inDestroying) {
                scheduleServiceRestartLocked(r, false);
            }
        }
    }

    // 如果是通过 bindService 来执行的，这里就会通知客户端.
    requestServiceBindingsLocked(r, execInFg);

    updateServiceClientActivitiesLocked(app, null, true);

    // If the service is in the started state, and there are no
    // pending arguments, then fake up one so its onStartCommand() will
    // be called.
    if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                null, null));
    }

    // 通知调用 onStartCommand 接口
    sendServiceArgsLocked(r, execInFg, true);

    if (r.delayed) {
        if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (new proc): " + r);
        getServiceMap(r.userId).mDelayedStartList.remove(r);
        r.delayed = false;
    }

    if (r.delayedStop) {
        // Oh and hey we've already been asked to stop!
        r.delayedStop = false;
        if (r.startRequested) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                    "Applying delayed stop (from start): " + r);
            stopServiceLocked(r);
        }
    }
}
```

上面代码的重点在于 `app.thread`, 这个 IApplicationThread 对象同样也是一个 Bindle 接口，与前文提交的 gDefault 不同之处在于两者的方向是相反的。前者是应用进程操作AMS，而后者则是AMS操作应用进程。IApplicationThread 对应的实现是 ApplicationThread，我们看看这个类是如何处理 `scheduleCreateService` 这个方法的。

```java
public final void scheduleCreateService(IBinder token,
        ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
    updateProcessState(processState, false);
    CreateServiceData s = new CreateServiceData();
    s.token = token;
    s.info = info;
    s.compatInfo = compatInfo;

    sendMessage(H.CREATE_SERVICE, s);
}
```

原来还是通过 mH Handler 这种方式来执行的呀，这与前面文章提及的知识是完全一样的。如果大家想详细地了解的话，建议看看这些系列文章 [Android 系统学习计划](http://www.woaitqs.cc/android/2016/06/16/android-system-learning)。在 Handler 中的 handleMessage 是如何处理 CREATE_SERVICE 这个消息的？

```java
case CREATE_SERVICE:
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceCreate");
    handleCreateService((CreateServiceData)msg.obj);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    break;
```

继续看 handleCreateService 方法的实现。

```java
private void handleCreateService(CreateServiceData data) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();

    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);
    Service service = null;
    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        service = (Service) cl.loadClass(data.info.name).newInstance();
    } catch (Exception e) {
        if (!mInstrumentation.onException(service, e)) {
            throw new RuntimeException(
                "Unable to instantiate service " + data.info.name
                + ": " + e.toString(), e);
        }
    }

    try {
        if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);

        ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
        context.setOuterContext(service);

        Application app = packageInfo.makeApplication(false, mInstrumentation);
        service.attach(context, this, data.info.name, data.token, app,
                ActivityManagerNative.getDefault());
        service.onCreate();
        mServices.put(data.token, service);
        try {
            ActivityManagerNative.getDefault().serviceDoneExecuting(
                    data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
        } catch (RemoteException e) {
            // nothing to do.
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(service, e)) {
            throw new RuntimeException(
                "Unable to create service " + data.info.name
                + ": " + e.toString(), e);
        }
    }
}
```

方法实现相对简单，首先通过 ClassLoader 加载相关的 class 对象，然后将 Service 和 Application 绑定在一起，最后调用 Service 的 onCreate 方法。到此为止，Service 就完全启动了。

如果使用 Service 的方式是 startService，那么在 Service 启动后，就可以执行 sendServiceArgsLocked 方法，从而在 Service 的 onStartCommand 里面可以执行相应的后台代码，需要特别说明的是，这个是执行在 UI 线程上的，因而建议不要执行耗时的任务，如果是耗时的任务，需要通过多线程的方式来避免主线程的阻塞。即使应用当前不在前台，阻塞 UI 线程，也是很不好的情况。

我们看看 sendServiceArgsLocked 是如何实现的。

```java
private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
        boolean oomAdjusted) throws TransactionTooLargeException {
    final int N = r.pendingStarts.size();
    if (N == 0) {
        return;
    }

    while (r.pendingStarts.size() > 0) {
        Exception caughtException = null;
        ServiceRecord.StartItem si;
        try {
            r.app.thread.scheduleServiceArgs(r, si.taskRemoved, si.id, flags, si.intent);
        } catch (TransactionTooLargeException e) {
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Transaction too large: intent="
                    + si.intent);
            caughtException = e;
        } catch (RemoteException e) {
            // Remote process gone...  we'll let the normal cleanup take care of this.
            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Crashed while sending args: " + r);
            caughtException = e;
        } catch (Exception e) {
            Slog.w(TAG, "Unexpected exception", e);
            caughtException = e;
        }

        if (caughtException != null) {
            // Keep nesting count correct
            final boolean inDestroying = mDestroyingServices.contains(r);
            serviceDoneExecutingLocked(r, inDestroying, inDestroying);
            if (caughtException instanceof TransactionTooLargeException) {
                throw (TransactionTooLargeException)caughtException;
            }
            break;
        }
    }
}
```

这里通过循环的方式，将在队列中的消息，依次通过 `app.thread` 发布到应用进程中去，如果中途发生了 TransactionTooLargeException 异常，则会提前终止这个过程。`app.thread` 的 scheduleServiceArgs 方法，也是通过 mH 这个 Handler 来执行的，发送的消息为 `SERVICE_ARGS`。

```java
case SERVICE_ARGS:
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceStart");
    handleServiceArgs((ServiceArgsData)msg.obj);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    break;
```

```java
private void handleServiceArgs(ServiceArgsData data) {
    Service s = mServices.get(data.token);
    if (s != null) {
        try {
            if (data.args != null) {
                data.args.setExtrasClassLoader(s.getClassLoader());
                data.args.prepareToEnterProcess();
            }
            int res;
            if (!data.taskRemoved) {
                res = s.onStartCommand(data.args, data.flags, data.startId);
            } else {
                s.onTaskRemoved(data.args);
                res = Service.START_TASK_REMOVED_COMPLETE;
            }

            QueuedWork.waitToFinish();

            try {
                ActivityManagerNative.getDefault().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
            } catch (RemoteException e) {
                // nothing to do.
            }
            ensureJitEnabled();
        } catch (Exception e) {
            if (!mInstrumentation.onException(s, e)) {
                throw new RuntimeException(
                        "Unable to start service " + s
                        + " with " + data.args + ": " + e.toString(), e);
            }
        }
    }
}
```

handleServiceArgs 方法中，`s.onStartCommand` 就是我们书写后台代码的地方，当这个方法执行完成后，又通过 gDefault 这个遥控器通知 AMS 来任务完成了。

对于 startService 这种方式而言，是需要手动调用 stopSelf 方法来结束 Service 的，背后的原理与 startService 方法类似，这里就不再赘述。

--------------------

### 0X01 bindService 调用流程

bindService 相较于 startService 要复杂一些，通过这种方式实现的 Service，容易多个组件绑定到它，通过 ServiceConnection 的方式来进行通信。当没有任何其他组件，连接到这个 Service 时，该 Service 会自动销毁。

bindService 方法是这样声明的。

```java
@Override
public boolean bindService(Intent service, ServiceConnection conn,
        int flags) {
    warnIfCallingFromSystemProcess();
    return bindServiceCommon(service, conn, flags, Process.myUserHandle());
}
```

```java
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
        UserHandle user) {
    // ignore some codes ...
    validateServiceIntent(service);
    try {
        // ignore some codes ...
        int res = ActivityManagerNative.getDefault().bindService(
            mMainThread.getApplicationThread(), getActivityToken(), service,
            service.resolveTypeIfNeeded(getContentResolver()),
            sd, flags, getOpPackageName(), user.getIdentifier());
        if (res < 0) {
            throw new SecurityException(
                    "Not allowed to bind to service " + service);
        }
        return res != 0;
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
}
```

可以看到同样是通过 gDefault 这个遥控器来通知 AMS 进行相应的操作的，原理与上面 startService 相同，接收到遥控器的指令后，ActiveServices 的 bindSericeLocked 方法开始执行。bindSericeLocked 在进行一些校验，确认进程创建成功等等步骤后，还是通过 `app.thread` 发送 `BIND_SERVICE` 消息，来执行对应的逻辑。

```java
private void handleBindService(BindServiceData data) {
    Service s = mServices.get(data.token);
    if (DEBUG_SERVICE)
        Slog.v(TAG, "handleBindService s=" + s + " rebind=" + data.rebind);
    if (s != null) {
        try {
            data.intent.setExtrasClassLoader(s.getClassLoader());
            data.intent.prepareToEnterProcess();
            try {
                if (!data.rebind) {
                    IBinder binder = s.onBind(data.intent);
                    ActivityManagerNative.getDefault().publishService(
                            data.token, data.intent, binder);
                } else {
                    s.onRebind(data.intent);
                    ActivityManagerNative.getDefault().serviceDoneExecuting(
                            data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                }
                ensureJitEnabled();
            } catch (RemoteException ex) {
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(s, e)) {
                throw new RuntimeException(
                        "Unable to bind to service " + s
                        + " with " + data.intent + ": " + e.toString(), e);
            }
        }
    }
}
```

在 onBind 方法，给调用者返回 Binder 对象，通过 publishService 方法通过到 AMS 内部去，我们看看接下来发生了什么。

```java
public void publishService(IBinder token, Intent intent, IBinder service) {
    // Refuse possible leaked file descriptors
    if (intent != null && intent.hasFileDescriptors() == true) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }

    synchronized(this) {
        if (!(token instanceof ServiceRecord)) {
            throw new IllegalArgumentException("Invalid service token");
        }
        mServices.publishServiceLocked((ServiceRecord)token, intent, service);
    }
}
```

ActiveServices 中的代码也相对简单, 遍历建立起的 ServiceConnection，并调用它们的 connected 方法，这也是我们需要编写后台代码的地方。

```java
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
    final long origId = Binder.clearCallingIdentity();
    try {
        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "PUBLISHING " + r
                + " " + intent + ": " + service);
        if (r != null) {
            Intent.FilterComparison filter
                    = new Intent.FilterComparison(intent);
            IntentBindRecord b = r.bindings.get(filter);
            if (b != null && !b.received) {
                b.binder = service;
                b.requested = true;
                b.received = true;
                for (int conni=r.connections.size()-1; conni>=0; conni--) {
                    ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
                    for (int i=0; i<clist.size(); i++) {
                        ConnectionRecord c = clist.get(i);
                        if (!filter.equals(c.binding.intent.intent)) {
                            continue;
                        }
                        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Publishing to: " + c);
                        try {
                            c.conn.connected(r.name, service);
                        } catch (Exception e) {
                            Slog.w(TAG, "Failure sending service " + r.name +
                                  " to connection " + c.conn.asBinder() +
                                  " (in " + c.binding.client.processName + ")", e);
                        }
                    }
                }
            }

            serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
        }
    } finally {
        Binder.restoreCallingIdentity(origId);
    }
}
```

### 0X02 总结

Service 作为四大组件之一，提供了不需要前台页面情况下，在后台继续执行任务的能力。Service 一般有两种使用方式，分别是通过 startService 和 bindService，前者适合执行一次性的任务，而后者则具备一定交互的能力，可以用作处理相对复杂的后台逻辑。

关于更多详细的用法，还是建议阅读官方教程，[Services](https://developer.android.com/guide/components/services.html).

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年9月20日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
