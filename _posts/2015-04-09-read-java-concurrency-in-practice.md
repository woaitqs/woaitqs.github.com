---
layout: post
title: "读薄《Java 并发实践》"
keywords : "源码, 并发实践, 详细, 同步, CompletionService"
description: "读薄《Java 并发实践》, 书中的关键点"
category: "java"
tags: [java, program, 并发]
---
{% include JB/setup %}

## 并发编程的目的

### 线程的优势
发挥多处理器的优势，异步事件的处理，高响应性

### 线程需要解决的问题
- 安全性问题（永远不发生糟糕的事情）
- 活跃性问题（某件正确的事情最终会发生）
- 性能问题 （线程创建和切换的开销）

------------

## 线程安全性

> 线程安全类：当多个线程访问某个类的时候，不管运行环境采用何种调度方式或者这些线程将如何交替执行，并且在主调代码中不需要任何额外的同步或协同，这个类都能表现出正确的行为

> 无状态的对象一定是线程安全的

### 原子性
#### 竞态条件与复合操作
由于不恰当的执行时序而出现不正确的结果

实例：错误的延迟初始化

``` java
@NotThreadSafe
public class LazyRace<T> {
	private T instance;
	public T getInstance() {
		// 判断和申明实例的复合操作不是原子的，可能引发问题
		if (instance == null) {
			instance = T.class.newInstance();
		}
		return instance;
	}
}
```
> 保证状态的一致性，就需要在单个原子操作中更新所有相关的状态变量

### java的基础锁机制
``` java
synchronized (lock){

}
```
synchronized 是可以重入的，换言之某个线程试图获得一个已经由自己持有的锁，那么这个请求就会成功。获得锁的基础单位是「线程」
对于可能被多个线程同时访问的可变状态变量，在访问它的时候都需要持有同一个锁，在这种情况下，我们称为状态变量是由这个锁来保护的。@GuardedBy

一种常见的加锁约定是，把所有的可变状态都封装在对象内部，并通过对象的内置锁对所有访问可变状态的代码路径进行同步，使得该对象不会发生并发访问。

------------

##对象的共享

### 可见性
在没有进行同步的情况下，编译器，处理器以及Java 运行时都可能对操作的顺序进行调整，在缺乏足够同步的多线程程序里面，想要对内存操作顺序进行判断，是不可能的

当一个线程修改了某些值，这个改变并不是马上对其他线程可见的。
> The Java Memory Model
A memory model describes when one thread's actions are guaranteed to be visible to another. The Java memory model (JMM) is quite an achievement: previously, memory models were specific to each processor architecture. A cross-platform memory model takes portability well beyond being able to compile the same source code: you really can run it anywhere. It took until Java 5 (JSR 133) to get the JMM right.
The JMM defines a partial ordering on program actions (read/write, lock/unlock, start/join threads) called happens-before. Basically, if action X happens-before Y, then X's results are visible to Y. Within a thread, the order is basically the program order. It's straightforward. But between threads, if you don't use synchronized or volatile, there are no visibility guarantees. As far as visible results go, there is no guarantee that thread A will see them in the order that thread B executes them. Brian even invoked special relativity to describe the disorienting effects of relative views of reality. You need synchronization to get inter-thread visibility guarantees.

java的内存模型要求，变量的读取和写入是原子的，除了非volatitle类型的long和double。
> 加锁的意义不仅在于互斥行为，还包括内存可见性。

### volatile
仅当Volatile变量能简化代码的实现以及同步策略的验证时，才应该使用他们。

1. 对变量的写入操作不依赖与变量的当前值，或者只有单个线程在更新这个值
2. 在访问变量时不需要加锁
3. 该变量不与其他状态变量一起纳入不变性条件里面

### 发布与逸出
发布 - 对象能够在当前作用域之外的代码里面使用
#### this 逸出

```java
public class ThisEscape {
	Public ThisEscape(EventSource source) {
		source.registeListener {
			// 发布EventListener时，也隐含发布了ThisEscape实例本身
			new EventListener() {
				public void onEvent(Event e) {
					doSomething(e);
				}
			}
		}
	}
}
```

#### 线程封闭
所有操作都在同一个线程里面完成，这个技术称为线程封闭(Thread Confinement)
栈封闭 - 线程内部使用或者局部使用
ThreadLocal - 使线程中的某个值与保存的对象关联起来

#### 不变性
不可变对象一定是线程安全的，Java内存模型里面，final域能够确保初始化过程的安全性。除非某个域是可变的，否则应该将他声明为Final

#### 安全发布
```java
public class UnSafeProgram {
	public Holder holder;

	public void init() {
		holder = new Holder(34);
	}
}

public class Holder {  
    private int n;  

    public Holder(int n) {
    	// Object会先于子类的构造函数执行，赋予默认值。
        this.n = n;
    }  

    public void assertSanity() {  
        if (n != n) {
        	// 除了发布的线程外，其他线程看到的Holder域是一个失效值，因此可能看到一个空引用
        	// 或者之前的旧值。
            throw new AssertionError("This statement is false.");  
        }
    }  
}
```

由于不可变对象是一种非常安全的对象，因此java内存模型为不可变对象的共享提供了一个特殊的初始化安全性保证。即便某个对象的引用对其他线程是可见的，也不意味着对象状态对使用该对象的线程一定是可见的。

1. 在静态初始化函数中初始化一个对象引用
2. 将对象的引用保存在volatile或者AtomicReferance对象中
3. 保存在某个正确构建对象的Final类型域中
4. 将对象的引用保存在由一个锁保护的域中

------------

## 对象的组合

### JAVA监视器模式
java 为每一个Object都关联着一种锁(Mutex)，通过这个锁来保证在Sync模块内，同时只有一个线程在执行代码

## 基础构建模块

并发容器与同步工具类

## 结构化并发应用程序

### Executor框架

### 为何用使用Executor框架

1. 线程生命周期的开销非常高
2. 资源消耗
3. 稳定性

### 框架介绍

java类库中，任务执行的框架主要是Thread，而是Executor。

```java
public interface Executor {
	void execute(Runnable command);
}
```

Executor 是基于生产者-消费者模式，提交任务的操作相当于生产者（生产待完成的工作单元），执行任务的线程相当于消费者（执行完这些工作单元）。需要实现生产者-消费者模式，最简单地就是使用Executor

#### 常见的线程池

```java
newFixedThreadPool
newCachedThreadPool
newSingleThreadPool
newScheduledThreadPool
```

Executor 只有在所有非守护线程全部终止后才会退出，为了解决这种生命周期问题，Executor扩展了ExecutorService接口，添加了一些用于管理生命周期的方法。

一般用ScheduledThreadPool来代替Timer来执行一些与时间相关的任务。

#### 携带结果的任务Callable与Future

Runnable 是一种很有局限性的抽象，Callable是一种更好的抽象，任务主入口有一个返回值，并可能抛出一个异常。相对于ExecutorService，Future是一个针对于任务生命周期的抽象。

```java
public interface Callable<V> {
	V call() throws Exception;
}
```

[Feature Interface](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Future.html)

#### 使用CompletionService来进行任务处理

Omitting many details:

1. ExecutorService = incoming queue + worker threads
2. CompletionService = incoming queue + worker threads + output queue

### 后续的内容将陆续以专门文章的形式推出，请期待
