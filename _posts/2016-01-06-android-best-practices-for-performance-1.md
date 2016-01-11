---
layout: post
title: "Android Best Practices for Performance (1)"
description: "Managing Your App's Memory"
category: "program, java, android"
tags: [android, program]
---
{% include JB/setup %}

RAM 在任何软件开发中都是非常珍贵的资源，在内存受限的手机系统中显得更加弥足珍贵。尽管Android的Davlik虚拟机能够对内存自动进行垃圾回收，但这并不意味着你能忽略你的应用对内存的分配和释放。

为了使得垃圾回收器能够及时回收内存，需要避免内存泄漏（例如一个全局变量持有对你的引用），同时要保证在合适的时间释放掉这些引用对象。对于大多数应用而言，做到上面的内容就足够了，Davlik负责处理剩下的内容。

接下来的内容将说明Android系统如何管理 app 进程，内存分配，和如何在开发 Android 应用的时候降低内存使用。当需要得知一些如何在 Java 编程中合理利用内存资源，可以参考一些网上的其他书籍。Android 系统提供了比较方面的方式便于你去分析你应用的内存占用情况，链接在[Investigating Your RAM Usage](http://developer.android.com/intl/zh-tw/tools/debugging/debugging-memory.html)

## Android 如何管理内存

Android系统并未给内存提供交互区，但还是使用了分页和mmapping的技术去管理内存。这也就意味着任何你修改的内存（无论是给新对象分配内存，还是引用了mmapped的内容）都会在RAM中被保存下来。因而完全释放内存的唯一方法就是释放你所持有的对象，使得内存能够被垃圾回收器回收。

### 共享内存

为了适应RAM中的一些需求，Android 系统尝试通过共享内存的方式来实现跨进程操作。

1. 没有一个应用进程都是从已经存在的 Zygote 进程中fork出来的。Zygote 进程在系统启动、加载FrameWork 层代码和相应资源的时候启动。 当需要开启一个新的应用进程的时候，系统会 fork Zygote 进程，然后将应用程序跑在这个新的进程上面。这就是使得绝大多数Framework层的代码和资源能够被所有应用进程使用到。

2. 大多数静态数据都被 mmapped 到一个进程上面。这不仅仅使得同样的数据能够在进程间进行共享，而且使得它能够在需要的时候被paged out。常见的static数据包括Dalvik Code，app resources，so文件等。

3. 大多数情况下，Android通过显式的分配共享内存区域(例如ashmem或者gralloc)来实现动态RAM区域能够在不同进程之间进行共享的机制。例如，Window Surface在App与Screen Compositor之间使用共享的内存，Cursor Buffers在Content Provider与Clients之间共享内存。

因为大量共享内存的使用，对于 App 所使用的内存资源就需要进行深思熟虑。

### 分配和回收内存资源

下面是一些关于 Android 系统如何分配和回收内存的一些Tips

1. 每个进程的 Davlik 堆被指定到一块虚拟机内存大小。这个大小定义了堆大小，能够根据需要进行扩容（但是存在一个上限，这个上限由系统进行定义）

2. 逻辑上讲的Heap Size和实际物理意义上使用的内存大小是不对等的，Proportional Set Size(PSS)记录了应用程序自身占用以及和其他进程进行共享的内存。

### 限制应用内存

为了支持多任务同时执行，Android对每个App设置了堆大小限制，准确的堆大小限制在不同机型上有不同的的表现。如果你的App分配了超出限制的 App 堆大小，就会收到 OutOfMemoryError.

在某些机型上，需要通过查询在当前设备商有多少可以内存大小，来决定缓存多少数据在内存中。可以通过查询 getMemoryClass() 来获取相应的信息。这个方法会返回当前可以堆大小。[这里有更详细的介绍](http://developer.android.com/intl/zh-tw/training/articles/memory.html#CheckHowMuchMemory)

### 应用切换

当用户切换的时候，Android 系统通过 LRU 的方式来对进程进行缓存。例如，当用户第一次启动了一个应用，系统会为此创建一个进程，当用户离开这个app的时候，进程不会立即被回收。系统缓存了这个进程，所以当用户回到这个应用的时候，进程能够被重新应用，以快速启动。

如果你的应用进程已经被缓存起来了，并且持有了暂时不被使用的内存(这部分内存不被用户所使用，而且会对系统的运行性能造成影响)。所以，当系统运行内存较少的时候，系统会回收掉最近没有使用的进程，即便这样对于某些内存敏感的进程还是会有些特殊照顾的。如何使得进程更长久的存在，将在后续的章节中说明。

关于进程和线程的说明，点击[这里](http://developer.android.com/intl/zh-cn/guide/components/processes-and-threads.html)