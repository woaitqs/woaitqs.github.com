---
layout: post
title: "和Android Memory谈一场不分手的恋爱"
description: "定位内存泄露问题，内存抖动问题，ArrayMap优化原理，Bitmap优化原理"
keywords: "Memory, 内存优化, 定位泄露, ArrayMap, Bitmap"
category: "android"
tags: [android, memory, performance]
---
{% include JB/setup %}

爱情大多数时候都是美好而甜蜜的，但也时常让我们烦恼，内存也是这样的，自动垃圾回收使得我们不用管内存的分配和释放，但稍微不注意，可能就掉进坑里面了。这边文章将主要围绕 Android Memory的各个方面进行展开，让我们知道如何与 Memory 谈恋爱，并尽可能地甜蜜。

<!--break-->

----------

## 一、待嫁闺中

内存是一个有着沉鱼落雁面容，和体贴善良的女子，她对你含情脉脉的眼神，让你心动流连(妈蛋又跑题了)。Android会给每个应用程序分配适当大小的线程，这样想象如果不加以限制，势必会影响到其他程序和系统本身的稳定性。内存主要是分为两部分，堆和栈(实际上还有方法区，常量池等)。具体可参看[《深入理解java虚拟机》](https://book.douban.com/subject/24722612/)

栈一般与线程相关，JVM(Davlik/Art)在创建每一个线程的时候，会分配一定的栈空间给线程，栈中存储这基本类型数据，以及对象引用等，这些部分主要用于解决程序的运行问题，这就好比内存妹子的灵魂。

相对而言，堆则用来解决程序的数据存储问题，这就是内存的肉身。如果想要知道妹子有多重，告诉你一个秘诀, 通过`getMemoryClass`方法可以得知。堆是程序主要进行分配和释放的地方，因而这一块需要良好的管理。下面介绍Android如何进行垃圾回收。

----------


## 二、举案齐眉

爱情是一门付出与回报的行为艺术，两者总是相辅相成的。而Android内存模型，通过自动垃圾回收的方式，帮我们完成了这一过程。我们使用Android内存模块来获取内存，然后内存回收模块回收掉不再使用的内存。

### 堆的分区

年轻代：新创建的对象都存放在这里。因为大多数对象很快变得不可达，所以大多数对象在年轻代中创建，然后消失。当对象从这块内存区域消失时，我们说发生了一次“minor GC”。
年老代：经过几次收集，寿命不断延长，一段时间后依然存活的对象。
永久代：主要存放加载的Class类级对象如class本身，method，field。

### 如何知道对象不再使用？

早期的垃圾回收采用引用计数(reference counting)的机制。每个对象包含一个计数器。当有新的指向该对象的引用时，计数器加1。当引用移除时，计数器减1。当计数器为0时，认为该对象可以进行垃圾回收。但这样存在的问题，在于循环引用上面，如果A引用B，B引用A，即便两者都不再使用，也无法释放内存。

为了解决对象循环引用这个问题，后续采用了更为准确的对象遍历方式。如下图所示，垃圾回收器会建立有向图的方式进行内存管理，通过GC Roots来往下遍历，当发现有对象出于不可达状态的时候，就会对其标记为不可达，以便于后续的GC回收。
![GC遍历](http://img.blog.csdn.net/20150627115915868)

### GC的时机

1. 调用函数dvmHeapSourceAlloc在Java堆上分配指定大小的内存。如果分配成功，那么就将分配得到的地址直接返回给调用者了。函数dvmHeapSourceAlloc在不改变Java堆当前大小的前提下进行内存分配，这是属于轻量级的内存分配动作。
2. 如果上一步内存分配失败，这时候就需要执行一次GC了。不过如果GC线程已经在运行中，即gDvm.gcHeap->gcRunning的值等于True，那么就直接调用函数dvmWaitForConcurrentGcToComplete等到GC执行完成就是了。否则的话，就需要调用函数gcForMalloc来执行一次GC了，参数False表示不要回收软引用对象引用的对象。
3. GC执行完毕后，再次调用函数dvmHeapSourceAlloc尝试轻量级的内存分配操作。如果分配成功，那么就将分配得到的地址直接返回给调用者了。
4. 如果上一步内存分配失败，这时候就得考虑先将Java堆的当前大小设置为Dalvik虚拟机启动时指定的Java堆最大值，再进行内存分配了。这是通过调用函数dvmHeapSourceAllocAndGrow来实现的。
5. 如果调用函数dvmHeapSourceAllocAndGrow分配内存成功，则直接将分配得到的地址直接返回给调用者了。
6. 如果上一步内存分配还是失败，这时候就得出狠招了。再次调用函数gcForMalloc来执行GC。参数true表示要回收软引用对象引用的对象。
7. GC执行完毕，再次调用函数dvmHeapSourceAllocAndGrow进行内存分配。这是最后一次努力了，成功与事都到此为止。

总结的说，GC发生一般发生在两种情况下。第一种情况是没有足够内存分配请求的分存时，会调用Heap类的成员函数CollectGarbageInternal触发一个原因为kGcCauseForAlloc的GC。第二种情况下分配出请求的内存之后，堆剩下的内存超过一定的阀值，就会调用Heap类的成员函数RequestConcurrentGC请求执行一个并行GC


----------


## 三、面有难色

感情总不会是一帆风顺，友谊的小船也常常说翻就翻。这不内存妹子就发起了小脾气，心情有些抖动，这是一种我们称之为「内存抖动」的现象。

在极短的时间内，分配大量的内存，然后又释放它，这种现象就会造成内存抖动。典型的情况是，在View控件的onDraw方法里分配大量内存，又释放大量内存，这种做法极易引起内存抖动，从而导致性能下降。因为onDraw里的大量内存分配和释放会给系统堆空间造成压力，触发GC工作去释放更多可用内存，而GC工作起来时，又会吃掉宝贵的帧时间 (帧时间是 16ms)，最终导致性能问题。

![绘制超时](http://hukai.me/images/gc_overtime.png)

![内存抖动示例](https://raw.githubusercontent.com/kamidox/blogs/master/images/memory_churn.gif)

<p style="color:red">这个地方还需要完善</p>


----------


## 四、心存芥蒂
爱情并不是在任何时候都那么美丽，也时常闹矛盾。内存是个敏感的姑娘，你对她不精心的小伤害，也往往让她对你心存芥蒂，久而久之可能最后分崩离析。在进行的开发的过程中，要注意哪些忽略她感受的小细节，这些细节让她对你的安全感下降很多。我们称这种现象为「内存泄露」。

Android内存泄漏指的是进程中某些对象（垃圾对象）已经没有使用价值了，但是它们却可以直接或间接地引用到 gc roots 导致无法被GC回收。无用的对象占据着内存空间，使得实际可使用内存变小，形象地说法就是内存泄漏了。

### 泄漏有哪些危害

运行性能的问题: Android在运行的时候，如果内存泄露导致其他组件可用的内存变少，一方面会使得GC的频率加剧，在发生GC的时候，所有进程都必须进行等待，GC的频率越多，从而用户越容易感知到卡顿。另一方面，内存变少，将可能使得系统会额外分配给你一些内存，而影响整个系统的运行状况。

运行崩溃问题: 一旦内存不足以分配某些内存，那么将会导致崩溃，这对于体验而言是致命的。我们在进行内存分析的时候，可以发现总有一些机型会出现OutOfMemory的崩溃栈，大抵都和内存泄露有关。

### 泄漏示例

来看看下面这个例子，DataContainer 负责抓取和更新数据，我们可以通过 `register(DataListener listener)`方法来对数据更新进行关注。`MainActivity`就关注了DataListener，因而当数据发生回调的时候，`MainActivity`就能立刻做出相应。

其实这里发生了比较严重的泄漏情况，在启动`MainActivity`的时候，`MainActivity` 就会将 `DataListener` 注入到`DataContainer`里，而在`MainActivity`退出的时候，即使`MainActivity`不再使用，但是由于其被`DataContainer`所引用，导致`MainActivity`无法被回收，将会一直占据着内存，影响程序的运行。

```java
public class DataContainer {
  private List<DataListener> dataListeners;
  private static DataContainer instance;

  static {
    instance = new DataContainer();
  }

  private DataContainer() {
    dataListeners = new ArrayList<>();
  }

  public static DataContainer getInstance() {
    return instance;
  }

  /**
   * 监听数据变化.
   */
  public void register(DataListener listener) {
    dataListeners.add(listener);
  }

  public interface DataListener {
    void onDataChanged();
  }

}

public class MainActivity extends AppCompatActivity implements DataContainer.DataListener {

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    LeakContainer.getInstance().register(MainActivity.this);
  }

  @Override
  public void onDataChanged() {
    Toast.show
    // do nothings.
  }
}
```

### 定位泄漏的方法

女生的小情绪，往往难以琢磨，如果我们忽略这些小脾气，往往又会得到惩罚。因而我们需要寻找一种方法来定位可能的对于内存妹纸的伤害。我们将分别从两个方面，来帮助我们分析和定位。一是宏观方法，通过一些很简单的方法来判断是否存在泄露，另一方面是通过精确定位的方式来提出具体的解决方案。

#### Android Studio Memory Monitor

Android Studio 提供了非常方便的工具，便于我们定位问题。Android Monitors模块中含有 Memory Tab，这个Tab以流线的方式，展示了每一时刻内，已分配的内存和还空闲的内存。

图中浅蓝颜色的部分表示已经分配的内存，而灰色部分表明空闲的内存。
![gettingstarted_image003.png](https://ooo.0o0.ooo/2016/04/01/56fe26ccc38ea.png)
总选中Devices和相应的包名后，就能看到动态内存分配的情况。
![gettingstarted_image002.png](https://ooo.0o0.ooo/2016/04/01/56fe26cccacd4.png)
如下图所示，当已分配的内存剧烈下降的时候，就标明发生了GC事件，GC发生的时刻和频率是我们关注的重点。
![gettingstarted_image005.png](https://ooo.0o0.ooo/2016/04/01/56fe26ccd17d5.png)
我们回到刚才的那个例子 #泄漏示例# ，下图是点击`MainActivity`然后按返回键退出，再进入再退出，重复几次后，内存Monitor显示的结果。从这个例子中，我们可以看到，尽管进过了几次 `GC`,但是内存用量却一直在增大，说明有些对象被某些静态或者其他GC Roots的对象引用着，导致其不能被释放。因而可以说明，其存在比较严重的内存泄漏问题。
![内存泄露](https://ooo.0o0.ooo/2016/03/31/56fdeba581676.png)

#### Android Devices Monitor

Android Devices Monitor提供了比较方便辅助的定位方法，在 `Heap` Tab下面，显示着`% Used`的使用量，如果这个值在`GC`后没有明显下降，那么就意味着发生了内存泄漏，具体的操作步骤如下。

1. 选择DDMS视图，并打开Devices视图和Heap视图
2. 点击选择要监控的进程，比如：上图中我选择的是system_process
3. 选中Devices视图界面上的`update heap`图标
4. 点击Heap视图中的`Cause GC` 按钮（相当于向虚拟机发送了一次GC请求的操作）

第一次点击`Cause GC`后的内存占比
![QQ20160401-1.png](https://ooo.0o0.ooo/2016/04/01/56fe7a65a5289.png)
在多次退出和进入后的内存占比
![QQ20160401-2.png](https://ooo.0o0.ooo/2016/04/01/56fe7a65f0c28.png)

#### 精确定位方法

我们在查看是否存在内存泄漏情况的时候，基于的基础单位往往是Activity，因而就可以想到一种思路，即通过在界面回退后，强制进行GC，然后判断是否还存在对该Activity的引用，这样就能得知是否存在泄漏。

![使用Monitor进行定位](https://ooo.0o0.ooo/2016/02/27/56d15d70346dc.png)

[MAT 使用简介](http://www.lightskystreet.com/2015/09/01/mat_usage/)

具体的的实施步骤如下：

1. 客户端中打开相应的Activity，并执行可能触发内存泄漏的操作.
2. 退出Activity界面，并点击Initiate GC(左起第二个按钮)
3. 点击Dump Java Heap，等待一会后，这个时候可以看到Dump 出来的日志。
4. 由于Android Profile文件不被 MAT 支持，因为我们需要执行转换操作。 ./hprof-conv path/file.hprof exitPath/heap-converted.hprof
5. 在 MAT 中打开文件，并选择Leak Suspects Report,等待最后的结果。
6. `Select * From instanceof android.app.Activity` 通过Activity的类名来过滤信息，在右键菜单里面，分别点击Merge Paths to Shortest GC Root 和 exclude all phantom/weak/soft etc. references, 排除被弱引用持有的情况。

#### LeakCanary 自动定位

Square 开源了LeakCanary来用作对于内存泄露情况的自动检测。

LeakCanary实现了引用观察者RefWatcher。RefWatcher.watch() 创建一个 KeyedWeakReference 到要被监控的对象。通过在Activity重要的生命周期中，在后台线程检查引用是否被清除，如果没有，调用GC。如果在GC后，引用还是未被清除，那么可能发生了内存泄露，这时候把heap内存dump到 APP 对应的文件系统中的 .hprof 文件中。在另外一个进程中的 HeapAnalyzerService 有一个 HeapAnalyzer 使用HAHA来解析这个文件。得益于唯一的 reference key, HeapAnalyzer 找到 KeyedWeakReference，定位内存泄露。HeapAnalyzer 计算 到 GC roots 的最短强引用路径，并确定是否是泄露。如果是的话，建立导致泄露的引用链。引用链传递到 APP 进程中的 DisplayLeakService， 并以通知的形式展示出来。

可以看到，Square在使用LeakCanary并进行相应的修改后，效果还是相当不错的。
![内存泄漏比例](https://corner.squareup.com/images/leakcanary/oom_rate.png)
由于官方开源的LeakCanary只能在Debug版上使用，在Release上通过NullObject方式实现了一个空实现，来避免性能问题。如果想通过小流量的方式来批量地发现用户内存泄露的情况，那么就需要对源码进行整改，刚好我做了这么一件事情，有兴趣的人可以拿去使用。[移除UI展示等逻辑后可在Release上使用的LeakCanary](https://github.com/woaitqs/leakcanary_without_notification)。有了用户相关的数据的泄露栈就能很好地处理各种泄露问题，使得应用良好稳定地运行。

###  常见内存泄漏CASE与修复方法

#### 泄漏CASE

1. 注册对象未反注册
在组件启动后，注册了某个对象的观察者，在组件回收的时候，忘记取消注册了。可以参考这样的例子，Activity声明的时候实现了对于下载进度接口的监听，而这个监听接口在实现的时候使用的是强引用，如果不进行主动反注册，Activity会因为被下载库持有引用，从而导致无法回收。
2. 长线执行的异步任务
组件内部有一个可能长时间执行的任务，通过内部类持有了对组件的引用。想象这样一个场景，界面上的某一个组件需要异步地去请求天气数据，在得到结果后显示在界面上。在网络回调的Callback中，持有了这个组件，从而在网络请求执行过程中，组件是无法进行回收的。
3. Android SDK的泄露
这类泄露一般不严重，不用特殊处理。比如TextLine.sCached对象会持有一个拥有三个TextLine的对象池，但TextLine的回收方法recycle处理得有bug，在android-5.1.0_r1修复了一部分，修复连接。其他的泄露地方可从这里看出一部分，SDK泄露统计。
4. 类的静态变量持有大数据对象
静态变量长期维持到大数据对象的引用，阻止垃圾回收。
5. 资源对象未关闭象
资源性对象如Cursor、File、Socket，应该在使用后及时关闭。未在finally中关闭，会导致异常情况下资源对象未被释放的隐患。
6. Handler 泄漏
Handler通过发送Message与主线程交互，Message发出之后是存储在MessageQueue中的，有些Message也不是马上就被处理的。在Message中存在一个target，是Handler的一个引用，如果Message在Queue中存在的时间越长，就会导致Handler无法被回收。如果Handler是非静态的，则会导致Activity或者Service不会被回收。handler在使用过后，在组件退出的时候没有处理这些handler。通过Handler post出去一个任务后，没有在最后调用removeCallbacks的接口，清除掉所有跟这个Runnable相关的message。

#### 修复方法

1. 尽量避免在组件内部使用内部类，内部的一些逻辑类可以使用Static的声明，避免持有对组件的引用。
2. 如果一定要持有内部类的引用，可以通过WeakReference来进行封装，这样可以缓解掉一些泄漏情况。
3. 对于Handler使用较多的情况，可以考虑使用WeakHandler
4. 正确关闭资源，对于使用了BraodcastReceiver，ContentObserver，File，游标 Cursor，Stream，Bitmap等资源的使用，应该在Activity销毁时及时关闭或者注销。
5. 在 Java 的实现过程中，也要考虑其对象释放，最好的方法是在不使用某对象时，显式地将此对象赋值为 null，比如使用完Bitmap 后先调用 recycle()，再赋为null,清空对图片等资源有直接引用或者间接引用的数组（使用 array.clear() ; array = null）等，最好遵循谁创建谁释放的原则。
6. 对 Activity 等组件的引用应该控制在 Activity 的生命周期之内； 如果不能就考虑使用 getApplicationContext 或者 getApplication，以避免 Activity 被外部长生命周期的对象引用而泄露。

------------------

## 五、如胶似漆

处理好爱情里面的小摩擦以后，现在我们来把和内存妹纸的恋爱进行到底，如何呵护好来之不易的感情。接下来的章节将围绕内存优化进行展开，这部分内容也对我们的开发很有帮助。

### 善用ArrayMap和SparseArray

HashMap 在我们的程序中经常出现，作为高效率存储和检索的容器被频繁使用。如果了解[HashMap 实现原理](http://woaitqs.github.io/program/2015/04/14/read-source-code-about-hashmap)的话，就知道这是一种空间换时间的实现方式，在客户端开发中由于内存受限，原来以空间换时间的方式也变得不太适合。为了解决HashMap更占内存的弊端，Android提供了内存效率更高的ArrayMap。它内部使用两个数组进行工作，其中一个数组记录key hash过后的顺序列表，另外一个数组按key的顺序记录Key-Value值，如下图所示：
![ArrayMap 原理图](http://img.ptcms.csdn.net/article/201508/12/55cafcd911daf.jpg)

当你想获取某个value的时候，ArrayMap会计算输入key转换过后的hash值，然后对hash数组使用二分查找法寻找到对应的index，然后我们可以通过这个index在另外一个数组中直接访问到需要的键值对。如果在第二个数组键值对中的key和前面输入的查询key不一致，那么就认为是发生了碰撞冲突。为了解决这个问题，我们会以该key为中心点，分别上下展开，逐个去对比查找，直到找到匹配的值。并且 ArrayMap 会采用动态数组的方式，始终使得内存占用控制在合理的范围内。

SparseArray 相对于ArrayMap做了进一步的细化，避免了对基础数据的装箱操作。系统提供了SparseBoolMap，SparseIntMap，SparseLongMap，LongSparseMap等容器。这些容器的使用场景也和ArrayMap一致，需要满足数量级在千以内，数据组织形式需要包含Map结构。

### 不要滥用Enum

在Android最开始的文档中，写着尽量避免使用Enum，认为使用 Enum 带来了不少的性能问题。不过Enum 相对于 int 而言，确实提供了不少的代码可读性，而且在后续的优化中Enum带来的影响也降低了不少。比如当我们发现有人在使用Enum.oridinal()这样的方法时，大概就可以说明Enum被滥用了。

Android Support提供了更好的方式，在语义限制和性能之间达到一个平衡，这就是[Android Animation](http://developer.android.com/reference/android/support/annotation/StringDef.html)

```java
@Retention(SOURCE)
  @StringDef({
     POWER_SERVICE,
     WINDOW_SERVICE,
     LAYOUT_INFLATER_SERVICE
  })
  public @interface ServiceName {}
  public static final String POWER_SERVICE = "power";
  public static final String WINDOW_SERVICE = "window";
  public static final String LAYOUT_INFLATER_SERVICE = "layout_inflater";
  ...
  public abstract Object getSystemService(@ServiceName String name);
```

### Bitmap 并不可怕

这一章节会介绍一些处理与加载Bitmap对象的常用方法，这些技术能够使得程序的UI不会被阻塞，并且可以避免程序超出内存限制。如果我们不注意这些，Bitmaps会迅速的消耗掉可用内存从而导致程序崩溃，出现下面的异常:java.lang.OutofMemoryError: bitmap size exceeds VM budget.

在Android应用中加载Bitmaps的操作是需要特别小心处理的，有下面几个方面的原因:

1. 移动设备的系统资源有限。Android设备对于单个程序至少需要16MB的内存。Android Compatibility Definition Document (CDD), Section 3.7. Virtual Machine Compatibility 中给出了对于不同大小与密度的屏幕的最低内存需求。 应用应该在这个最低内存限制下去优化程序的效率。当然，大多数设备的都有更高的限制需求。
2. Bitmap会消耗很多内存，特别是对于类似照片等内容更加丰富的图片。 例如，Galaxy Nexus的照相机能够拍摄2592x1936 pixels (5 MB)的图片。 如果bitmap的图像配置是使用ARGB_8888 (从Android 2.3开始的默认配置) ，那么加载这张照片到内存大约需要19MB(2592*1936*4 bytes) 的空间，从而迅速消耗掉该应用的剩余内存空间。
3. Android应用的UI通常会在一次操作中立即加载许多张bitmaps。 例如在ListView, GridView 与 ViewPager 等控件中通常会需要一次加载许多张bitmaps，而且需要预先加载一些没有在屏幕上显示的内容，为用户滑动的显示做准备。

[高效加载大图](http://hukai.me/android-training-course-in-chinese/graphics/displaying-bitmaps/load-bitmap.html)

### 善用Service资源

如果你的应用需要在后台使用service，除非它被触发并执行一个任务，否则其他时候service都应该是停止状态。另外需要注意当这个service完成任务之后因为停止service失败而引起的内存泄漏。

当你启动一个service，系统会倾向为了保留这个service而一直保留service所在的进程。这使得进程的运行代价很高，因为系统没有办法把service所占用的RAM空间腾出来让给其他组件，另外service还不能被paged out。这减少了系统能够存放到LRU缓存当中的进程数量，它会影响app之间的切换效率。它甚至会导致系统内存使用不稳定，从而无法继续保持住所有目前正在运行的service。

限制你的service的最好办法是使用IntentService， 它会在处理完交代给它的intent任务之后尽快结束自己。更多信息，请阅读Running in a Background Service.

当一个Service已经不再需要的时候还继续保留它，这对Android应用的内存管理来说是最糟糕的错误之一。因此千万不要贪婪的使得一个Service持续保留。不仅仅是因为它会使得你的应用因为RAM空间的不足而性能糟糕，还会使得用户发现那些有着常驻后台行为的应用并且可能卸载它。

------------------

## 参考文献

1. [http://www.cnblogs.com/vamei/archive/2013/04/28/3048353.html](http://www.cnblogs.com/vamei/archive/2013/04/28/3048353.html)
2. [http://daily.zhihu.com/story/7364069](http://daily.zhihu.com/story/7364069)
3. [http://droidyue.com/blog/2014/11/02/note-for-google-io-memory-management-for-android-chinese-edition/](http://droidyue.com/blog/2014/11/02/note-for-google-io-memory-management-for-android-chinese-edition/)
