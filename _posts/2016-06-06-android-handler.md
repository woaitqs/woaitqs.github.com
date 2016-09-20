---
layout: post
title: "Android Handler机制全解析"
description: "详细说明Android Handler机制，以及Looper，MessageQueue，Handler在里面扮演的角色"
keywords: "Android, Handler, 机制, 实现原理"
category: "android"
tags: [android, program, handler]
---
{% include JB/setup %}

### 生产者与消费者
端午节将至，大家可能已经安排好自己的行程，不久就将出发，有做飞机前往目的地，也有做轮渡在近海游玩。设想，我们做飞机出游，只需按时抵达机场，在等候一段时间，自然有相应的飞机带我们前往心怡许久的地方。

![transport.jpg](http://o8p68x17d.bkt.clouddn.com/QQ201606138.png)

仔细想想，你不需要关心是哪一趟航班将承担此次的出行任务，另一方面，出行的航班也不关心会有哪些旅客将要登记。互相不知道细节，却能彼此很好的协作，这就是 [生产者-消费者](https://program-think.blogspot.com/2009/03/producer-consumer-pattern-0-overview.html) 带来的好处。

<!--break-->

生产者-消费者模式，在实际开发中极为常见，源于其主要的两个好处。

- 解耦。这一点是核心好处，生产者和消费者完全不用知道彼此的实现细节，未尝有利于独立模块的实现。
- 并发支持。这一点在处理耗时任务时，也经常被用到。生产者和消费者可以保持不同的频率，可以单独调整，以满足实际的需要。

### 缓冲区
如果生产者在完成任务后，立即交给消费者，那么两者之间势必是耦合的，这和普通的函数调用没有什么区别。实现解耦，就得引入第三方，生产者和消费者都和这个缓冲区打交道，而彼此互不知道，就类似于房产中介。

这个缓冲区才如何实现呢？由于涉及到并发的问题，这个缓冲区必须是线程安全的，生产者和消费者都需要同时访问，生产者往这个区域写入内容，而消费者从这个区域里读取内容。可以将机场理解为缓冲区，乘客涌入机场，而飞机将乘客从机场带入目的地。

对于这个缓冲区，并没有什么特别的要求，只需要实现`put(T item)` 和 `T get()` 两个接口即可。java中常用 [BlockingQueue](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/BlockingQueue.html) 作为缓冲区。

------------------

### Handler、Looper 和 MessageQueue机制讲解

在起初接触Android的时候，第一次用于做异步通信的方式，很可能就是 Hnadler 机制，其实从某种意义上而言，这种机制也是基于 生产者-消费者 模式展开的。例如UI线程就是消费者，在其他线程（生产者）上通过 Handler 将要执行的 Callback ，迁移到 UI 线程上执行。

[MessageQueue](https://developer.android.com/reference/android/os/MessageQueue.html) 就是前文中提及的缓冲区，这里是Android Framework对其的特殊实现。而生产者需要将任务提交给缓冲区，而这个提交工作是由 [post](https://developer.android.com/reference/android/os/Handler.html#post(java.lang.Runnable))([Runnable](https://developer.android.com/reference/java/lang/Runnable.html) runable) 或者 [postDelayed](https://developer.android.com/reference/android/os/Handler.html#postDelayed(java.lang.Runnable, long))([Runnable](https://developer.android.com/reference/java/lang/Runnable.html) r, long delayMillis) 等post方法来执行。而消费者（主要是UI thread）通过 [looper](https://developer.android.com/reference/android/os/Looper.html) 不断地从 MessageQueue 中取出任务再执行。

简要的示意图如下：

![Handler框架](http://o8p68x17d.bkt.clouddn.com/QQ20160613-9.png)

***********************

#### Message 的定义
既然是使用的生产者-消费者模式，那么生产和消费的内容是什么了？答案就是 Message。现在看看 Message 中几个常用的变量。

```java
/**
 * User-defined message code so that the recipient can identify
 * what this message is about. Each {@link Handler} has its own name-space
 * for message codes, so you do not need to worry about yours conflicting
 * with other handlers.
 */
public int what;

/**
 * arg1 and arg2 are lower-cost alternatives to using
 * {@link #setData(Bundle) setData()} if you only need to store a
 * few integer values.
 */
public int arg1;

/**
 * arg1 and arg2 are lower-cost alternatives to using
 * {@link #setData(Bundle) setData()} if you only need to store a
 * few integer values.
 */
public int arg2;

/*package*/ Bundle data;

/*package*/ Handler target;

/*package*/ Runnable callback;
```

这里 what 类似于标明 Message 的类型 Id，调用者可以通过这个 what 做出相应的逻辑调整。arg1 arg2 以及后面的 object 是用作额外数据传输的。 而 target 则定义了是哪一个消费者来处理哪一个 callback。为什么要使用一个 Target 变量来标明是哪一个消费者了？因为一个 LooperThread 是允许存在多个 Handler 的，也就是多个消费者，而这些消息都被放置到一个 MessageQueue 队列中，target 就起到了区别它们的目的。callback 即实际要执行的东西。

Message 同时提供了 obtain() 方法，不推荐使用 new Message() 的方法，而是重复回收利用 Message，和 ThreadPool 的原理类似。

#### MessageQueue

Android中的 MessageQueue 就是前文中提及的缓冲区，Android Framework 对其做了一些 JNI 的调用，来进行一些保护。这里的具体实现就不提及了，只需要知道线程安全，并提供了 `Message next() ` 和 `boolean enqueueMessage(Message msg, long when)` 接口即可。

![MessageQueue](https://i-msdn.sec.s-msft.com/dynimg/IC709523.png)

#### Looper 是什么？

前文提及的是，Looper 主要负责的工作是从 MessageQueue 中取出要执行的任务，也就是维护一个消息循环，现在看看 Looper 具体是怎么运作的。

```java
class LooperThread extends Thread {
  public Handler mHandler;

  public void run() {
      Looper.prepare();

      mHandler = new Handler() {
          public void handleMessage(Message msg) {
              // process incoming messages here
          }
      };

      Looper.loop();
  }
}
```

这是常用的Looper 示例，通过`Looper.prepare()`进行相应的初始化工作，而`Looper.loop()`则正式开启消息循环。简单来说，Looper 使得一个普通的线程具备了消息循环的能力，也就是获取信息并消费的能力，现在从源码中简单分析下几个重要的方法。

```java
// 检查Looper是否创建，并保证其全局唯一性
private static void prepare(boolean quitAllowed) {
	if (sThreadLocal.get() != null) {
		throw new RuntimeException("Only one Looper may be created per thread");
	}
    // 通过ThreadLocal 关键字保证每一个线程只存在一份
	sThreadLocal.set(new Looper(quitAllowed));
}
```

```java
private Looper(boolean quitAllowed) {
    // 私有构造函数，初始化 MessageQueue.
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

loop方法在实现上也很简单，首先检查Looper创建，如果没有就抛出异常。这里的`Binder.clearCallingIdentity()`是移除旧有的 Binder Identity，并在每次循环中做检验，为什么要调用这个方法，可以参考这篇[博文](http://blog.csdn.net/windskier/article/details/6921672)，也推荐大家看我之前写的 [Binder 完全解析](http://www.jianshu.com/notebooks/4445374/latest) 。之后，进入消息循环，不断地从MessageQueue中获取要处理的消息，并通过 `msg.target.dispatchMessage(msg)` 方法进行消息派发。

```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        msg.target.dispatchMessage(msg);

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();

        msg.recycleUnchecked();
    }
}
```

### Handler

Handler 在系统中承担的角色较为复杂，可是当做是全局的操作者，接下来简要地进行下分析。

```java
public Handler(Callback callback, boolean async) {
    ...
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

Handler 必须依附于相应的Looper线程，如果线程没有Looper 或者 Looper 没有调用 prepare 方法，会抛出`new RuntimeException("Can't create handler inside thread that has not called Looper.prepare()")`的异常。道理也很简单，不开启相应的Looper，Handler 发送的消息往什么地方传递了？ 在这个构造函数里，赋值相应的 MessageQueue 和 callback。callback的定义如下，即在 Looper Thread 要执行的任务，一般情况可以是在其他线程耗时操作执行完成后，回到Looper Thread 上要执行的UI 更新操作。

```java
/**
 * Callback interface you can use when instantiating a Handler to avoid
 * having to implement your own subclass of Handler.
 *
 * @param msg A {@link android.os.Message Message} object
 * @return True if no further handling is desired
 */
public interface Callback {
    public boolean handleMessage(Message msg);
}
```

Handler 通过 post postDelayed 等等方法，来将相应的 Message 发送到消息队列中去，最后通过 sendMessageAtTime() 来进行发送，进行的工作特别简单，将 Message.target 指定为自己，同时将自己加入到队列中。

```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

##### 机制总结

- **Handler 消息处理者**
它主要有两大作用：① 处理Message。② 发送Message，并将某个Message压入到MessageQueue中。

- **Looper 轮询器**
在 Looper里面的 loop()函数中有个死循环,它不断地从 MessageQueue 中取出一个Message,然后传给Handler进行处理,如此循环往复。假如队列为空,那么它会进入休眠。

- **MessageQueue 消息队列**
消息队列中含有多个Message，每个Message中包含了具体的调用信息。

------------------

### Android 使用Handler实例

在每一个Application启动的时候，会给这个Application分配一个 ActivityThread ，就是我们所说的 UI 线程，一个类的入口方法是 main 函数，这里看下源码。

```java
public static void main(String[] args) {
    SamplingProfilerIntegration.start();

    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    CloseGuard.setEnabled(false);

    Environment.initForCurrentUser();

    // Set the reporter for event logging in libcore
    EventLogger.setReporter(new EventLoggingReporter());

    Security.addProvider(new AndroidKeyStoreProvider());

    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);

    Process.setArgV0("<pre-initialized>");

    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();
    thread.attach(false);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    if (false) {
        Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
    }

    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

这里使用的 MainLooper 中的 Handler 被称为 H，这里的 H 定义了一些列的消息，如下所示，也就是说 Activity 相关的线程间通信，就是依赖于 Handler 机制的。

```java
...

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

...
```

-----------
