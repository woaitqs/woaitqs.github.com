---
layout: post
title: "Android Binder 全解析(3) -- AIDL原理剖析"
description: "AIDL是如何基于Binder实现的，其原理是什么？"
keywords: "Android, Binder, 机制, 实现原理"
category: "android"
tags: [android, program, binder]
---
{% include JB/setup %}

<div style="border:solid 1.5px #ccc;padding:20px 20px 10px 20px;margin-bottom: 20px;border-radius: 6px;">

          <span style="position: relative;padding: 5px 15px 5px 15px;background-color: white;bottom: 30px;font-size: 16px;font-weight: bolder;">摘要</span>

          <span style="position: relative;display: block;bottom: 15px;">
          本文是 Android 系统学习系列文章中的第二章节，在前面一些细节概念的铺垫下，大体上知道 Binder Framework 是怎么运作的，在这边文章中，将详细说明下 Binder Framework 的具体实现，这一套机制如何盘活整个 Android 系统。对此系列感兴趣的同学，可以收藏这个链接 <a herf="http://www.woaitqs.cc/2016/06/16/android-system-learning.html">Android 系统学习</a>，也可以点击 <a href="http://www.woaitqs.cc/feed.xml">RSS订阅</a> 进行订阅。
          </span>
</div>

- **DONE:** [Android Binder 完全解析（一）概述](http://www.woaitqs.cc/android/2016/05/23/android-binder.html)
- **DONE:** [Android Binder 完全解析（二）设计详解](http://www.woaitqs.cc/android/2016/05/26/android-binder-token.html)
- **DONE:** [Android Binder 完全解析（三）AIDL实现原理分析](http://www.woaitqs.cc/android/2016/05/30/android-binder-proxy-and-token.html)

<!--break-->

### IBinder 与 Binder

一个在同进程的对象的抽象是 Object，但这个对象是不能被跨进程使用的，要想跨进程使用，在 Android 中就必须依附于 Binder Framework。基于抽象设计的原理，Android系统将一个可远程操作的应用定义为 [IBinder](https://developer.android.com/reference/android/os/IBinder.html)，在这个接口中定义了一个可远程调用对象应该具有的属性和方法，在代码中实际使用的Binder 也都是继承自 IBinder 对象。我们接下来分析 IBinder 中定义了那些接口，是怎么抽象出可远程调用相关的方法的。

```java
public interface IBinder {
    ... ...

    // 查看 binder 对应的进程是否存活
    public boolean pingBinder();
    // 查看 binder 是否存活，需要注意的是，可能在返回的过程中，binder 不可用
    public  boolean isBinderAlive();

    /**
     * 执行一个对象的方法，
     *
     * @param 需要执行的命令
     * @param 传输的命令数据，这里一定不能为空
     * @param 目标 Binder 返回的结果，可能为空
     * @param 操作方式，0 等待 RPC 返回结果，1 单向的命令，最常见的就是 Intent.
     */
    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException;

    // 注册对Binder死亡通知的观察者，在其死亡后，会收到相应的通知
    // 这里先跳过，后续
    public void linkToDeath(DeathRecipient recipient, int flags) throws RemoteException;

    ... ...

}
```

如注释中所述，最重要的方法是 transact 方法，要理解这个方法是干嘛的，得看 [`ipcthreadstate.cpp`](https://android.googlesource.com/platform/frameworks/native/+/37b4496/libs/binder/IPCThreadState.cpp) 源代码里的说明。

```c++
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck();
    flags |= TF_ACCEPT_FDS;

    ... ...

    if (err == NO_ERROR) {
        LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),
            (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }

    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }

    if ((flags & TF_ONE_WAY) == 0) {
        ... ...
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
        ... ...
    } else {
        err = waitForResponse(NULL, NULL);
    }

    return err;
}
```

这是截取过后的源码，刨除了一些支线逻辑，从这个代码里面可以看到的主要逻辑就是：根据Uid 和 Pid （用户id和进程id）进行相应的校验，校验通过后，将相应的数据写入 `writeTransactionData `，其后在 `waitForResponse ` 里面读取前面写入的值，并执行相应的方法，最后返回结果。

总结地来说，就是 IBinder 抽象了远程调用的接口，任何一个可远程调用的对象都应该实现这个接口。由于 IBinder 对象是一个高度抽象的结构，直接使用这个接口对于应用层的开发者而言学习成本太高，需要涉及到不少本地实现，因而 Android 实现了 Binder 作为 IBinder 的抽象类，提供了一些默认的本地实现，当开发者需要自定义实现的时候，只需要重写 Binder 中的
`protected boolean onTransact(int code, Parcel data, Parcel reply, int flags)` 方法即可。

-------

### 自动售货机的故事

先偏一个题，夏天一到，天气也变得炎热，地铁旁边的自动售货机开始有更多的人关顾了。那么售货机是怎么工作的了？通过对这个分析，可以对Binder Proxy/Stub 模式更好地理解。

和我们打交道得是售货机，而不是背后的零售商，道理很简单，零售商的位置是固定的，也就意味着有很大的交通成本，而和售货机打交道就轻松很多，毕竟售货机就在身边。因而从客户端的角度上看，只需要知道售货机即可。

再从零售商的角度来看，要和为数众多的售货机打交道也是不容易的事情，需要大量的维护和更新成本，如果将起交由另一个中介公司，就能够省心不少。零售商只关心这个中介公司，按时发货，检查营收，这就是所有它应该完成的工作。

![binder proxy / stub 结构](https://ooo.0o0.ooo/2016/05/30/574c286c7ddd2.png)

如上图所示，在 Binder Framework 中也采用了类似的结构，Proxy 就相当于前面提及的售货机，Stub 相当于这里的中介公司，在这样的设计下，客户端只需要和 Proxy 打交道，服务端只需要和 Stub 打交道，调理清晰很多。这种模式又被称为 Proxy / Stub 模式，这种模式也值得我们在日后的开发中借鉴。另外需要注意的是，为了开发的需要，通常 Proxy 和 Stub 实现了相同的接口。

在这里 Stub 是具体的远程方法实现，也被称为`存根`，Proxy 继承了相应的接口，但只是在这里面调用远程方法，并返回相应的结果。

### AIDL 是什么？它是如何运作的？

虽然上述的模式帮助我们将远程方法调用与服务具体实现解耦开来，但是这里面还是又不少代码需要实现，而且远程方法调用这一块应该是重复代码，那么有什么方法来帮我们简化这个步骤吗？

Android 的设计者在一开始也意识到这个问题，推出了 [AIDL](https://developer.android.com/guide/components/aidl.html)，如下图所示

![AIDL](https://ooo.0o0.ooo/2016/05/30/574c3164ecdef.png)

现在我们来分析下，Android Framework 是如何把 AIDL 运行起来的，首先来看看一个大数相乘的例子，想把这个耗时的操作，放到另一个进程上来调用，于是定义了下面的 IMultiply.aidl 文件。

```java
// IMultiply.aidl
package com.wandoujia.baymax.aidl;

interface IMultiply {
    long multiply(long left, long right);
}
```

在 Android Studio 中点击 Clean / Rebuild Project  之后，可以在 build / genreated / Source 的目录里面找到 IMultiply.java 文件，具体的代码如下：

```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: /Users/QisenTang/Program/Wandoulabs/baymax/packages/baymax/src/main/aidl/com/wandoujia/baymax/aidl/IMultiply.aidl
 */
package com.wandoujia.baymax.aidl;
// Declare any non-default types here with import statements

public interface IMultiply extends android.os.IInterface {
  /**
   * Local-side IPC implementation stub class.
   */
  public static abstract class Stub extends android.os.Binder implements com.wandoujia.baymax.aidl.IMultiply {
    private static final java.lang.String DESCRIPTOR = "com.wandoujia.baymax.aidl.IMultiply";

    /**
     * Construct the stub at attach it to the interface.
     */
    public Stub() {
      this.attachInterface(this, DESCRIPTOR);
    }

    /**
     * Cast an IBinder object into an com.wandoujia.baymax.aidl.IMultiply interface,
     * generating a proxy if needed.
     */
    public static com.wandoujia.baymax.aidl.IMultiply asInterface(android.os.IBinder obj) {
      if ((obj == null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin != null) && (iin instanceof com.wandoujia.baymax.aidl.IMultiply))) {
        return ((com.wandoujia.baymax.aidl.IMultiply) iin);
      }
      return new com.wandoujia.baymax.aidl.IMultiply.Stub.Proxy(obj);
    }

    @Override
    public android.os.IBinder asBinder() {
      return this;
    }

    @Override
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
      switch (code) {
        case INTERFACE_TRANSACTION: {
          reply.writeString(DESCRIPTOR);
          return true;
        }
        case TRANSACTION_multiply: {
          data.enforceInterface(DESCRIPTOR);
          long _arg0;
          _arg0 = data.readLong();
          long _arg1;
          _arg1 = data.readLong();
          long _result = this.multiply(_arg0, _arg1);
          reply.writeNoException();
          reply.writeLong(_result);
          return true;
        }
      }
      return super.onTransact(code, data, reply, flags);
    }

    private static class Proxy implements com.wandoujia.baymax.aidl.IMultiply {
      private android.os.IBinder mRemote;

      Proxy(android.os.IBinder remote) {
        mRemote = remote;
      }

      @Override
      public android.os.IBinder asBinder() {
        return mRemote;
      }

      public java.lang.String getInterfaceDescriptor() {
        return DESCRIPTOR;
      }

      @Override
      public long multiply(long left, long right) throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        long _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          _data.writeLong(left);
          _data.writeLong(right);
          mRemote.transact(Stub.TRANSACTION_multiply, _data, _reply, 0);
          _reply.readException();
          _result = _reply.readLong();
        } finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }
    }
    static final int TRANSACTION_multiply = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
  }

  public long multiply(long left, long right) throws android.os.RemoteException;
}
```

这段自动的生成的代码，可读性上不是很好，但不影响我们进行分析。这段自动生成的代码中，最重要的是三个部分，`Proxy`，`Stub`和`asInterface`。

首先看看`Proxy`的实现，首先是将远程的Binder作为参数传入进来，再来看看 multiply 这个方法里面的下面几个步骤。首先是将left 和 right的参数写入到_data中去，同时在 远程binder 调用结束后，得到返回的 _reply ，在没有异常的情况下，返回_reply.readLong()的结果。根据 [Android Binder 完全解析（一）概述](http://www.jianshu.com/p/c11333e77910) 提到的内容，这里可以粗略地将 Binder 看做远程服务抛出的句柄，通过这个句柄就可以执行相应的方法了。这里需要额外说明的是，写入和传输的数据，都是 Parcelable，Android Framework 中提供的

```java
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeLong(left);
_data.writeLong(right);
mRemote.transact(Stub.TRANSACTION_multiply, _data, _reply, 0);
_reply.readException();
_result = _reply.readLong();
```

再来看看`Stub`里面的实现，`Stub`本身继承了Binder抽象类，本身将作为一个句柄，实现在服务端，但传递给客户端使用。同样也看看 `onTransact`里面的方法，首先是通过 `long _arg0; _arg0 = data.readLong();` 读取 left 和 right 参数，在`this.multiply(_arg0, _arg1)`计算过后，将结果写入到reply中。

细心的读者，可以从上面的描述中，发现一些有意思的东西。Proxy 是写入参数，读取值；Stub 是读取参数，写入值。正好是一对，那因此我们是不是可以做出这样的论断呢？Proxy 和 Stub 操作的是一份数据？恭喜你，答对了。

在前文提及的内容里，用户空间和内核空间是互相隔离的，客户端和服务端在同一用户空间，而 Binder Driver 在内核空间中，常见的方式是通过 copy_from_user 和 copy_to_user 两个系统调用来完成，但 Android Framework 考虑到这种方式涉及到两次内存拷贝，在嵌入式系统中不是很合适，于是 Binder Framework 通过 ioctl 系统调用，直接在内核态进行了相关的操作，节省了宝贵的空间。

可能大家也注意点 `Proxy` 这里是private权限的，外部是无法访问的，但这里是 Android 有意为之，抛出了 `asInterface` 方法，这样避免了对 `Proxy`可能的修改。

### 系统使用AIDL的场景

AIDL 被Android 系统广泛使用，在许多地方都能看到相应的场景，这里以 [电话服务](https://github.com/android/platform_frameworks_base/blob/master/telephony/java/com/android/internal/telephony/ITelephony.aidl) 作为例子，来简单说明下如何在系统没有提供挂断电话的API的情况下强行挂电话。

通过的 ITelephony.aidl 的查看，可以在里面找到相应的接口，不过这个类是 internal 包里面的，应用层无法访问。那怎么来完成这个操作了？

```java
 /**
  * End call if there is a call in progress, otherwise does nothing.
  *
  * @return whether it hung up
  */
 boolean endCall();
```

首先在相同包名下申明同样的AIDL文件，再通过编译后，就会生成相应的 Stub 文件。其次通过反射拿到 TELEPHONY_SERVICE 的binder。这个 binder 就是可以操作另一个进程来挂断电话的句柄。将这个binder 作为 Proxy 的参数，并通过 asInterface 注入进去，从何获得相应的接口，最后调用 telephony.endCall() 即可完成操作。

```java
Method method = Class.forName("android.os.ServiceManager").getMethod("getService", String.class);
IBinder binder = (IBinder) method.invoke(null, new Object[]{TELEPHONY_SERVICE});
ITelephony telephony = ITelephony.Stub.asInterface(binder);
telephony.endCall();
```

### 参考文献

1. [https://developer.android.com/guide/components/aidl.html](https://developer.android.com/guide/components/aidl.html)
2. [http://8204129.blog.51cto.com/8194129/1357365](http://8204129.blog.51cto.com/8194129/1357365)
