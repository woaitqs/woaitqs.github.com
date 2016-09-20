---
layout: post
title: "Android Activity 生命周期是如何实现的"
description: "描述了 Activity 生命周期是如何实现的"
keywords: "android lifecycle 源码解析 binder activity"
category: "android"
tags: [android, program, activity]
---
{% include JB/setup %}

<div style="border:solid 1.5px #ccc;padding:20px 20px 10px 20px;margin-bottom: 20px;border-radius: 6px;">

          <span style="position: relative;padding: 5px 15px 5px 15px;background-color: white;bottom: 30px;font-size: 16px;font-weight: bolder;">摘要</span>

          <span style="position: relative;display: block;bottom: 15px;">
          本文是 Android 系统学习系列文章中的第三章节的内容，在前面的文章 <a herf="http://www.woaitqs.cc/android/2016/06/21/activity-service.html">Android 应用进程启动流程</a> 讲了 Android 是如何启动的，在这篇文章里，将详细说明 Activity 生命周期的实现原理，onCreate、onResume、onPause 等主要生命周期回调是如何实现的，ActivityManangerService 在里面扮演的角色。对此系列感兴趣的同学，可以收藏这个链接 <a herf="http://www.woaitqs.cc/2016/06/16/android-system-learning.html">Android 系统学习</a>，也可以点击 <a href="http://www.woaitqs.cc/feed.xml">RSS订阅</a> 进行订阅。
          </span>
</div>

<!--break-->

### 背景知识介绍

这篇文章中涉及到 Binder 的设计实现，如果对这一块不是很熟悉的话，建议阅读后续代码时，先看看下面的链接的内容，特别是 AIDL 的实现。

- [Android Binder 完全解析（一）概述](http://www.woaitqs.cc/android/2016/05/23/android-binder.html)
- [Android Binder 完全解析（二）设计详解](http://www.woaitqs.cc/android/2016/05/26/android-binder-token.html)
- [Android Binder 完全解析（三）AIDL实现原理分析](http://www.woaitqs.cc/android/2016/05/30/android-binder-proxy-and-token.html)

* ActivityThread: 运行在应用进程的主线程上，响应 ActivityManangerService 启动、暂停Activity，广播接收等消息。
* ActivityThread.mH: 继承自 Handler，用于发送组件相关的事件消息。
* ActivityThread.ApplicationThread: Binder 对象, 通过 ApplicationThreadProxy 的构造函数注入进去，作为 ActivityThread 的内部类，响应具体的 ActivityManagerService 的方法请求。
* ApplicationThreadProxy: Binder Proxy 对象，传递到 ActivityManagerService 中，当 ActivityManagerService 需要让 Activity 做相应诸如启动、销毁等事情时，会通过 ActivityThread.ApplicationThread 执行具体的业务逻辑。
* Instrumentation: Android 提供的用于监听 Activity 与系统交互的机制。
* ActivityManagerService: Android 核心的组件服务，用于管理控制各类组件。

![From Activity 和 To Activity](http://o8p68x17d.bkt.clouddn.com/from_to.png)

从实际的例子出发，会更方便理解，这里以从列表页面（以下简称 From Activity）跳转到详情页面（以下简称 To Activity）为例，说明在这个过程中发生的事情。

------------------------

### Activity 生命周期实现

#### 1）From Activity 请求 ActivityManageService 启动 To Activity

首先当点击 From Activity 中的列表页时，通过下面的代码发送 Intent，启动 To Activity。

```Java
Intent intent = new Intent(context, Detail.class);
intent.putInt(Const.REST_ID, id);
startActivity(intent);
```

这段代码会执行到 Activity 的 `startActivity` 方法，Activity 中也有多个不同签名的 `startActivity`，都会执行到 `startActivityForResult` 这个方法，扼要的代码如下:

```Java
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
  if (mParent == null) {
      Instrumentation.ActivityResult ar =
          mInstrumentation.execStartActivity(
              this, mMainThread.getApplicationThread(), mToken, this,
              intent, requestCode, options);
      if (ar != null) {
          mMainThread.sendActivityResult(
              mToken, mEmbeddedID, requestCode, ar.getResultCode(),
              ar.getResultData());
      }
      // ...
      cancelInputsAndStartExitTransition(options);
  } else {
      if (options != null) {
          mParent.startActivityFromChild(this, intent, requestCode, options);
      } else {
          // Note we want to go through this method for compatibility with
          // existing applications that may have overridden it.
          mParent.startActivityFromChild(this, intent, requestCode);
      }
  }
}
```

无论 Parent 是否为空，最后都是通过 Instrumentation 来执行启动的任务，Instrumentation 是 Android 设计者提供出来的中间层，方便开发者监听系统各种与 Application 的交互，同时也对测试工作很有帮助，接下来看看 Instrumentation 的具体代码。

Instrumentation 在提供相应的监听逻辑后，调用了 ActivityManagerNative 来执行相应的逻辑，如下所示。

```Java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    ...
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess();
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, , null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

这里先看看 `getDefault` 的实现，看看里面返回的是什么？

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

这里看到返回的是 ServiceManager.getService，ServiceManager 在 Framework 充当了 DNS 的作用，通过 ServiceManager 可以找到其他的系统服务，这里返回的正是名为的 activity 的 服务。在前面提供的链接 [Android Binder 完全解析（三）AIDL实现原理分析](http://www.woaitqs.cc/android/2016/05/30/android-binder-proxy-and-token.html) 中，也讲到了 ServiceManager 的实现，会更详细些。

接下来看看 asInterface 的实现，这不正是很标准的 AIDL 实现吗？传入的 obj 对象正是鼎鼎大名的 `ActivityManageService`, 这里通过 ActivityManagerProxy 进行代理，将接口暴露到应用层，这样对于调用出于 System_server 进程的方法时，就如同在本地调用一样。

```java
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

在 ActivityManagerProxy 中对 startActivity 进行转发的代码如下所示：

```java
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
```

这里采用的方式正是 Binder 的调用方式，在 ActivityManagerProxy 中持有了 ActivityManagerService 提供的 Binder 接口，所有的命令都是通过这个 Binder 进行消息发送的，在客户端进程调用 `startActivity` 时，会调用到 ActivityManagerService 的 `startActivity` 方法里面，在进行一些列的处理后，会执行到 ActivityStackSupervisor 的 `startActivityMayWait` 方法里面，继而调用到 `startActivityLocked` 方法中。

至此为止，在 From Activity 的启动 Activity 请求，已经传递到 `ActivityManagerService` 中。

#### 2）ActivityManagerService 响应启动请求

首先调用到 startActivity 方法里面。

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle options) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
        resultWho, requestCode, startFlags, profilerInfo, options,
        UserHandle.getCallingUserId());
}
```

其后进入到 startActivityAsUser 方法里面，在这个方法中，将逻辑移交到 `ActivityStackSupervisor` 中去。

```java
@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
    enforceNotIsolatedCaller("startActivity");
    userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
            false, ALLOW_FULL_ONLY, "startActivity", null);
    // TODO: Switch to user app stacks here.
    return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, options, false, userId, null, null);
}
```

这里的 mStackSupervisor 就是 ActivityStackSupervisor , 这个类在 Android 4.4 以后才引入的，用于管理 ActivityStack。对于前面举得这个例子，方法的运行 Trace 会沿着下面的方法来执行。

>
ActivityManagerService.startActivity()
ActvityiManagerService.startActivityAsUser()
ActivityStackSupervisor.startActivityMayWait()
ActivityStackSupervisor.startActivityLocked()
ActivityStackSupervisor.startActivityUncheckedLocked()
ActivityStackSupervisor.startActivityLocked()
ActivityStackSupervisor.resumeTopActivitiesLocked()
ActivityStackSupervisor.resumeTopActivityInnerLocked()

里面的逻辑非常复杂，也不在今天主要讲的范围内，这里只做简单介绍。

```java
final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
            Bundle options, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask) {
            ...
            int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                    componentSpecified, null, container, inTask);
            ...
            return res;
    }
```

在 startActivityMayWait 中，会去查看是否存在相应的Activity，如果同时存在多个相应的 Activity，弹出一个对话框，让用户进行选择。

```java
final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
        int err = ActivityManager.START_SUCCESS;
        ...
        err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);
        ...
        return err;
    }
```

在 startActivityLocked 和 startActivityUncheckedLocked 方法里, 建立了相应的 ActivityRecord 对象，加入到 TaskRecord 中，并根据相应的 LaunchMode 和 Flag 进行相应的进栈出栈处理，所有的细节都在这两个方法里面，就不在展开了。关于 Activity 的栈，这里有一篇很好的文章，可供翻阅，[Understand Android Activity's launchMode: standard, singleTop, singleTask and singleInstance](https://inthecheesefactory.com/blog/understand-android-activity-launchmode/)，还有一个不错的 PPT，讲解关于 LaunchMode 中的一些坑，[Manipulating Android tasks and back stack](http://www.slideshare.net/RanNachmany/manipulating-android-tasks-and-back-stack)。

最后执行到 startActivityLocked，调用 resumeTopActivitiesLocked 方法。

```java
final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {
        // ...
        if (doResume) {
            mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
        }
    }
```

经过一些方法的参数处理后，执行 resume 代码逻辑，在 resumeTopActivityInnerLocked 方法里。

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {
        ...
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
        }        ...
        return true;
    }
```

在启动下一个 Activity，需要将上一个 Activity 暂停掉，这在面试中也经常问到，大家可以注意下。上面的代码也也验证了先行 pause 的逻辑，在下面的章节中看看是如何暂停 From Activity的。

#### 3）暂停 From Activity

```java
final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping, boolean resuming,
            boolean dontWait) {
        ...
        if (prev.app != null && prev.app.thread != null) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
            try {
                EventLog.writeEvent(EventLogTags.AM_PAUSE_ACTIVITY,
                        prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName);
                mService.updateUsageStats(prev, false);
                prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                        userLeaving, prev.configChangeFlags, dontWait);
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }
        ...
    }
```

startPausingLocked 的方法里面调用了 prev.app.thread.schedulePauseActivity 这个方法，其中这里的 thread，是上文中传递过去的 IApplicationThread，这里指的是 ApplicationThreadProxy 对象。这里和前面调用 ActivityManageService 的方式相同，也是采用的 Binder 框架，将要具体的数据通过 Parcable 传递到 ActivityThread.ApplicationThread 中，接下来看看 ActivityThread 中 schedulePauseActivity的具体实现。

```java
public final void schedulePauseActivity(IBinder token, boolean finished,
                boolean userLeaving, int configChanges, boolean dontReport) {
            sendMessage(
                    finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                    token,
                    (userLeaving ? 1 : 0) | (dontReport ? 2 : 0),
                    configChanges);
        }
```

这里的 sendMessage 方法，是通过 ActivityThread 的 mH 这个类来进行 Message 的发放，这里传递的消息是 PAUSE_ACTIVITY_FINISHING，我们看看 ActivityThread 是如何处理这个消息的。

```java
case PAUSE_ACTIVITY_FINISHING:
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");
    handlePauseActivity((IBinder)msg.obj, true, (msg.arg1&1) != 0, msg.arg2,
            (msg.arg1&1) != 0);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    break;
```

逻辑还藏在 handlePauseActivity 里面，就接着往下看。

```java
private void handlePauseActivity(IBinder token, boolean finished,
        boolean userLeaving, int configChanges, boolean dontReport) {
    ActivityClientRecord r = mActivities.get(token);
    if (r != null) {
        //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
        if (userLeaving) {
            performUserLeavingActivity(r);
        }
        r.activity.mConfigChangeFlags |= configChanges;
        performPauseActivity(token, finished, r.isPreHoneycomb());
        // Make sure any pending writes are now committed.
        if (r.isPreHoneycomb()) {
            QueuedWork.waitToFinish();
        }
        // Tell the activity manager we have paused.
        if (!dontReport) {
            try {
                ActivityManagerNative.getDefault().activityPaused(token);
            } catch (RemoteException ex) {
            }
        }
        mSomeActivitiesChanged = true;
    }
}
```

```java
final Bundle performPauseActivity(ActivityClientRecord r, boolean finished,
            boolean saveState) {
    // ...
    mInstrumentation.callActivityOnPause(r.activity);
    // ...
    return !r.activity.mFinished && saveState ? r.state : null;
}
```

又来到了 Instrumentation 这个类里面，前文也提到这个类监听着应用与系统发生的通信，看来它还是挺尽职的。

```java
public void callActivityOnPause(Activity activity) {
    activity.performPause();
}
```

这里执行的方法是 Activity.performPause，这应该就是这次 Pause 流程的终点了。

```java
final void performPause() {
    mDoReportFullyDrawn = false;
    mFragments.dispatchPause();
    mCalled = false;
    onPause();
    mResumed = false;
    if (!mCalled && getApplicationInfo().targetSdkVersion
            >= android.os.Build.VERSION_CODES.GINGERBREAD) {
        throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onPause()");
    }
    mResumed = false;
}
```

果然，我们熟悉的 onPause 回调就放在这里，到此 Pause 过程就完成了，现在接着回到 handlePauseActivity 里面去，还有方法没有执行完成。

```java
private void handlePauseActivity(IBinder token, boolean finished,
        boolean userLeaving, int configChanges, boolean dontReport) {
    ActivityClientRecord r = mActivities.get(token);
    if (r != null) {
        // ...
        if (!dontReport) {
            try {
                ActivityManagerNative.getDefault().activityPaused(token);
            } catch (RemoteException ex) {
            }
        }
        mSomeActivitiesChanged = true;
    }
}
```

在执行完成 Pause 过程后，继续通过 ActivityManagerNative 执行 ActivityManagerService 的 activityPause 方法，这个机制与前面的过程相同。从这里可以看到，ActivityThread 和 ActivityManagerService 互相持有着双方的把柄，不，是 Binder 对象，Binder 机制使得在跨进程调用的时候，和同进程的方法调用没有什么区别。


#### 4）启动 To Activity

```java
@Override
public final void activityPaused(IBinder token) {
    final long origId = Binder.clearCallingIdentity();
    synchronized(this) {
        ActivityStack stack = ActivityRecord.getStackLocked(token);
        if (stack != null) {
            stack.activityPausedLocked(token, false);
        }
    }
    Binder.restoreCallingIdentity(origId);
}
```

Activity 的 pause 方法，也是通过 Stack 来完成，这样便于进行栈管理，来看看具体的实现。

```java
final void activityPausedLocked(IBinder token, boolean timeout) {
        //...
            if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSED: " + r
                    + (timeout ? " (due to timeout)" : " (pause complete)"));
            completePauseLocked(true);
        //...
}

private void completePauseLocked(boolean resumeNext) {
    //...
    if (resumeNext) {
        final ActivityStack topStack = mStackSupervisor.getFocusedStack();
        if (!mService.isSleepingOrShuttingDown()) {
            mStackSupervisor.resumeTopActivitiesLocked(topStack, prev, null);
        } else {
            mStackSupervisor.checkReadyForSleepLocked();
            ActivityRecord top = topStack.topRunningActivityLocked(null);
            if (top == null || (prev != null && top != prev)) {
                // If there are no more activities available to run,
                // do resume anyway to start something.  Also if the top
                // activity on the stack is not the just paused activity,
                // we need to go ahead and resume it to ensure we complete
                // an in-flight app switch.
                mStackSupervisor.resumeTopActivitiesLocked(topStack, null, null);
            }
        }
    }
    //...
}
```

接下来继续经过如下的调用栈

>
resumeTopActivitiesLocked
ActivityStack.resumeTopActivityLocked()
resumeTopActivityInnerLocked
startSpecificActivityLocked

最后调到了 startSpecificActivityLocked 方法里面来。

```java
void startSpecificActivityLocked(ActivityRecord r,
        boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
            r.info.applicationInfo.uid, true);
    r.task.stack.setLaunchTime(r);
    if (app != null && app.thread != null) {
        try {
            if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                    || !"android".equals(r.info.packageName)) {
                // Don't add this if it is a platform component that is marked
                // to run in multiple processes, because this is actually
                // part of the framework so doesn't make sense to track as a
                // separate apk in the process.
                app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                        mService.mProcessStats);
            }
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

这个方法存在两个分支，当对应的应用进程存在时，直接执行 realStartActivityLocked 方法，否者就先新建应用进程。关于新建应用进程的事宜，在我的这篇博客[http://www.woaitqs.cc/android/2016/06/21/activity-service.html](http://www.woaitqs.cc/android/2016/06/21/activity-service.html)里面，说得相对详细，这里就不再详细说明了，其实在进程启动完毕后，最后也会调用到 realStartActivityLocked 到这个方法里面来。

```java
app.thread.scheduleLaunchActivity(new Intent(r.intent), r,  
    System.identityHashCode(r),  
    r.info, r.icicle, results, newIntents, !andResume,  
    mService.isNextTransitionForward());  
```

在 realStartActivityLocked 方法中，同样也是通过 ApplicationThreadProxy 进行远程方法调用，最后会执行到 ActivityThread 的scheduleLaunchActivity 的方法中来，在下面的代码里面，可以看出 Binder 调用。

```java
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,  
        ActivityInfo info, Bundle state, List<ResultInfo> pendingResults,  
        List<Intent> pendingNewIntents, boolean notResumed, boolean isForward)  
        throws RemoteException {  
    Parcel data = Parcel.obtain();  
    data.writeInterfaceToken(IApplicationThread.descriptor);  
    intent.writeToParcel(data, 0);  
    data.writeStrongBinder(token);  
    data.writeInt(ident);  
    info.writeToParcel(data, 0);  
    data.writeBundle(state);  
    data.writeTypedList(pendingResults);  
    data.writeTypedList(pendingNewIntents);  
    data.writeInt(notResumed ? 1 : 0);  
    data.writeInt(isForward ? 1 : 0);  
    mRemote.transact(SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION, data, null,  
        IBinder.FLAG_ONEWAY);  
    data.recycle();  
}
```

和 Pause 的过程相同，只是这里发送的消息是 LAUNCH_ACTIVITY，我们看看在这一块是如何处理的。

```java
case LAUNCH_ACTIVITY: {
    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

    r.packageInfo = getPackageInfoNoCheck(
            r.activityInfo.applicationInfo, r.compatInfo);
    handleLaunchActivity(r, null);
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
```

handleLaunchActivity 的具体实现如下：

```java
private final void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {  
    //......  

    Activity a = performLaunchActivity(r, customIntent);  

    if (a != null) {  
        r.createdConfig = new Configuration(mConfiguration);  
        Bundle oldState = r.state;  
        handleResumeActivity(r.token, false, r.isForward);  

        //......  
    } else {  
        //......  
    }  
}  
```

这里有两个重要的方法，分别是 performLaunchActivity 和 handleResumeActivity 方法。

先看看 performLaunchActivity 方法，这里由几个重要的部分组成。

收集相应的组件信息。

```java
ComponentName component = r.intent.getComponent();
if (component == null) {
    component = r.intent.resolveActivity(
        mInitialApplication.getPackageManager());
    r.intent.setComponent(component);
}

if (r.activityInfo.targetActivity != null) {
    component = new ComponentName(r.activityInfo.packageName,
            r.activityInfo.targetActivity);
}
```

通过上面得到的组件信息，得到要具体启动的 Activity，这里的 Activity 是通过反射构建出的对象。

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
```

如果没有相应的 Application 信息，则会先建立相应的 Application。

```java
Application app = r.packageInfo.makeApplication(false, mInstrumentation);  
```

接着把 Activity 对象，attach 到相应的运行环境上面去。

```java
activity.attach(appContext, this, getInstrumentation(), r.token,  
r.ident, app, r.intent, r.activityInfo, title, r.parent,  
r.embeddedID, r.lastNonConfigurationInstance,  
r.lastNonConfigurationChildInstances, config);  
```

最后，又是通过 Instrumentation 调用了 Activity 的 onCreate 方法。

```java
mInstrumentation.callActivityOnCreate(activity, r.state);  
```

到此为止，Activity 完成了启动。

#### 5）Activity 的 onResume 方法，和 onDestroy 方法

>
ActivityStack.resumeTopActivityLocked()
ActivityStack.resumeTopInnerLocked()
IApplicationThread.scheduleResumeActivity()
ActivityThread.scheduleResumeActivity()
ActivityThread.sendMessage()
ActivityTherad.H.sendMessage()
ActivityThread.H.handleMessage()
ActivityThread.H.handleResumeMessage()
Activity.performResume()
Activity.performRestart()
Instrumentation.callActivityOnRestart()
Activity.onRestart()
Activity.performStart()
Instrumentation.callActivityOnStart()
Activity.onStart()
Instrumentation.callActivityOnResume()
Activity.onResume()

>
Activity.finish()
ActivityManagerNative.getDefault().finishActivity()
ActivityManagerService.finishActivity()
ActivityStack.requestFinishActivityLocked()
ActivityStack.finishActivityLocked()
ActivityStack.startPausingLocked()

onResume 回调和当 Activity 销毁时的 onDestroy 回调都与前面的过程大同小异，这里就只列举相应的方法栈，不再继续描述。

------------------------

### 总结

可以看到，Activity 的生命周期并不是一个类就可以简单完成的，在这其中需要多个模块之间进行通信合作，才能做到现在的效果。里面涉及到两种 Android 中常见的通信方式，分别是 Binder 机制和 Handler 机制。其中 Binder 机制用于 ActivityThread 和 ActivityManagerService 进行进程间通信，而 Handler 机制，则用在 ActivityThread 中，使得 ActivityThread 在合适的时机能处理相应的生命周期。

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年7月20日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
