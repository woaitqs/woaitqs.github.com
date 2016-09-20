---
layout: post
title: "Java HashMap 源码解析终极版"
description: "Source code about HashMap, hashmap 源码详细说明"
keywords : "源码, HashMap, 详细, 设计"
category: "program"
tags: [java, program]
---
{% include JB/setup %}

## 序言
ConCurrentHashMap 是一个被忽视的Java Concurrent包下面的类，在满足并发的「安全性」，和「活跃性」的前提下，做到了与不考虑线程安全的 HashMap 同等效率. 作者是大名鼎鼎的[Doug Lea](http://en.wikipedia.org/wiki/Doug_Lea)，他老人家在Java 并发领域做的贡献，确实是我们的榜样。下篇文章，对ConCurrentHashMap做一个分析，希望这个代码中的闪光点，能够对各位读者产生启发。这里先介绍HashMap做的实现，便于后面我们理解2者的差异，以及[Doug Lea]完成的ConCurrentHashMap类具有那些惊为天人的地方。

## JDK 是如何定义Map接口的
在设计一个通用的模块和功能的时候，我们需要静下心来分析下根本需求是什么？根据这个需求来建立我们的根本需求。

Map，就是Key-Value对，通过Key可以快速找到对应的Value，核心的需求是Put和Get方法。

```java
public void put(K,V);
public V get(K);
```
在实际的需求里面，虽然不同于Collection，但是一些基础功能和需求是共通的，所以需要额外地加上一些基础方法，比如`isEmpty()`,`size()`等。于是而后在原来的基础上添加了其他基础抽象，最后形成的接口大体可以如下

```java
public void clear();
public boolean containsKey(Object key);
public boolean containsValue(Object value);
public Set<Map.Entry<K,V>> entrySet();
public boolean equals(Object object);
...
public V remove(Object key);
public int size();

```

在这个时候，可能会有2个问题

1）为什么Map接口没有继承自Collection?

这个在JDK的问题里面所说的是Map，和Collection根本是两个东西，具体可以参考下面的引用
> This was by design. We feel that mappings are not collections and collections are not mappings. Thus, it makes little sense for Map to extend the Collection interface (or vice versa).
If a Map is a Collection, what are the elements? The only reasonable answer is "Key-value pairs", but this provides a very limited (and not particularly useful) Map abstraction. You can't ask what value a given key maps to, nor can you delete the entry for a given key without knowing what value it maps to.
Collection could be made to extend Map, but this raises the question: what are the keys? There's no really satisfactory answer, and forcing one leads to an unnatural interface.
Maps can be viewed as Collections (of keys, values, or pairs), and this fact is reflected in the three "Collection view operations" on Maps (keySet, entrySet, and values). While it is, in principle, possible to view a List as a Map mapping indices to elements, this has the nasty property that deleting an element from the List changes the Key associated with every element before the deleted element. That's why we don't have a map view operation on Lists.

Map可以用Collection来实现，但不一定意味着Map就一定是Collection，Collecton也不一定是Map。

2）为什么没有实现Iterator接口？

Map在定义上面就没有要求一定是可以迭代的，有人可能疑问用EntrySet来实现迭代啊？但是从设计的角度上去理解，如果用EntrySet的方式在接口里面申明Iterator接口，就意味着把内部的实现细节暴露出去了。所以宁愿提供`entrySet()`的接口。


```java
/**
 * Returns a {@code Set} containing all of the mappings in this {@code Map}. Each mapping is
 * an instance of {@link Map.Entry}. As the {@code Set} is backed by this {@code Map},
 * changes in one will be reflected in the other.
 *
 * @return a set of the mappings
 */
public Set<Map.Entry<K,V>> entrySet();
```

## Java是如何实现HashMap的

### HashMap 基础单元

HashMap作为我们经常用的类，几乎没有程序猿不熟悉Map的用法, 没有道理不去熟悉下HashMap的实现原理。

HashMap使用java提供的基础类型中的数组，用*HashMapEntry<K, V>[] table*来进行Key-Value键值对的存储。Hash需要完成的工作即是将Key通过Hash的算法，将Key hash到某一个小于数组长度的数值*i*上面，从而在这个位置*i*记录下对应的Value，亦即`table[i] = value`，这样调用者在调用`get(key)`方法时，先通过hash计算出i，调用`table[i]`可以迅速找到相应的Value，这是HashMap查询速度快的原因。

JDK首先构建了对基础单元的抽象-`HashMapEntry`，里面的核心成员变量如下代码，其中hash值一个用于缓存，便于进行比较，而next字段则实现了C++的指针，通过这个指针可以链式地找到下一位。这个链式结构的设计目的是为了解决Hash冲突的情况。

```java
final K key;
V value;
final int hash;
HashMapEntry<K, V> next;
```

### HashMap 如何避免冲突和解决冲突的

当两个Key同时hash到一个值时，就会出现这样的冲突。这个冲突主要有2种解决方法。

1. 开放地址，亦即如果hash冲突，则在空闲的位置进行插入
2. hash复用，同一个hash值，链式地加入多个value

HashMap通过递归，选择了第二种方式来解决冲突的问题。

整个HashMap的核心部分是hash方法，我们先从最核心的方法看起，``HashMap``是如何实现将hashCode值映射到table数组里面的索引里面的。

```java
@Override public V put(K key, V value) {
    if (key == null) {
        return putValueForNullKey(value);
    }

    int hash = Collections.secondaryHash(key);
    HashMapEntry<K, V>[] tab = table;
    int index = hash & (tab.length - 1);
    for (HashMapEntry<K, V> e = tab[index]; e != null; e = e.next) {
        if (e.hash == hash && key.equals(e.key)) {
            preModify(e);
            V oldValue = e.value;
            e.value = value;
            return oldValue;
        }
    }

    // No entry for (non-null) key is present; create one
    modCount++;
    if (size++ > threshold) {
        tab = doubleCapacity();
        index = hash & (tab.length - 1);
    }
    addNewEntry(key, value, hash, index);
    return null;
}
```
```java
// Collections.secondaryHash(key)方法里面使用了
int hash = Collections.secondaryHash(key);
int index = hash & (tab.length - 1);
```
在这里面可以看到Hash的代码主要是Wang/Jenkins hash方法，这个方法具有两个属性

1. 雪崩性（更改输入参数的任何一位，就将引起输出有一半以上的位发生变化）
2. 可逆性（input ==> hash ==> inverse_hash ==> input）

Collections.secondaryHash能够使得hash过后的值的分布更加均匀，尽可能地避免冲突，[具体原理点连接](http://burtleburtle.net/bob/hash/integer.html)，注意这里的前提是table的长度为2的幂次方，从构造函数对capacity的改造可以看出来。`hash & (tab.length - 1)` 当tab.length为2^n-1的时候，可以保证结果不大于tab.length。

```java
public HashMap(int capacity) {
    // ...
    if (capacity < MINIMUM_CAPACITY) {
        capacity = MINIMUM_CAPACITY;
    } else if (capacity > MAXIMUM_CAPACITY) {
        capacity = MAXIMUM_CAPACITY;
    } else {
        capacity = Collections.roundUpToPowerOfTwo(capacity);
    }
    makeTable(capacity);
    // ...
}
```

这里举一个例子，如果现在Table的容量是16，如果最后的结果是 **hash & (16-1)**，也就是 **hash & (00001111)**。现在我们试着对hash值是31(00011111), 63(00111111),95(01011111)进行操作，理论上映射到的index值应该是不尽相同的，然而实际的情况确实如下的情形：

31=00011111 ==& 00001111==> 1111=15

63=00111111 ==& 00001111==> 1111=15

95=01011111 ==& 00001111==> 1111=15

因此Collections.secondaryHash需要解决的问题，就是避免上面的情况，效果见下面的例子：

31=00011111 ==secondaryHash==> 00011110==& 00001110==> 14

63=00111111 ==secondaryHash==> 00111100==& 00001100==> 12

95=01011111 ==secondaryHash==> 01011010==& 00001010==> 10

### HashMap 如何解决容量不足的问题

前面解决了hash冲突的问题，那么现在如何解决容量不足的问题了。考虑这样的情况，如果用户大量地调用put方法，在这种情况下，如果容量不变，那么势必会出现大量的冲突，调用get方法时，可能需要很长长度的遍历才能得到答案，性能损失严重，因此，我们可以发现源码中有这样一种方法：

```java
if (size++ > threshold) {
	tab = doubleCapacity();
	index = hash & (tab.length - 1);
}
```

JDK给出的方案是扩容，doubleCapacity()方法，比如原来容量为16，当现在的使用量大于*DEFAULT_LOAD_FACTOR\*Capacity*，下次直接把容量扩展到32.
这段代码注解说的是这整个类的精华，我们必须好好研究下.
>> Rehash the bucket using the minimum number of field writes, and this is the most subtle and delicate code in the class.

```java
/**
 * Doubles the capacity of the hash table. Existing entries are placed in
 * the correct bucket on the enlarged table. If the current capacity is,
 * MAXIMUM_CAPACITY, this method is a no-op. Returns the table, which
 * will be new unless we were already at MAXIMUM_CAPACITY.
 */
private HashMapEntry<K, V>[] doubleCapacity() {
    HashMapEntry<K, V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        return oldTable;
    }
    int newCapacity = oldCapacity * 2;
    HashMapEntry<K, V>[] newTable = makeTable(newCapacity);
    if (size == 0) {
        return newTable;
    }

    for (int j = 0; j < oldCapacity; j++) {
        /*
         * Rehash the bucket using the minimum number of field writes.
         * This is the most subtle and delicate code in the class.
         */
        HashMapEntry<K, V> e = oldTable[j];
        if (e == null) {
            continue;
        }
        int highBit = e.hash & oldCapacity;
        HashMapEntry<K, V> broken = null;
        newTable[j | highBit] = e;
        for (HashMapEntry<K, V> n = e.next; n != null; e = n, n = n.next) {
            int nextHighBit = n.hash & oldCapacity;
            if (nextHighBit != highBit) {
                if (broken == null)
                    newTable[j | nextHighBit] = n;
                else
                    broken.next = n;
                broken = e;
                highBit = nextHighBit;
            }
        }
        if (broken != null)
            broken.next = null;
    }
    return newTable;
}
```
我们注意其中这部分的代码

```java
int highBit = e.hash & oldCapacity;
newTable[j | highBit] = e;
```
我们现在来说明，oldCapacity假设为16(00010000), **int highBit = e.hash & oldCapacity**能够得到高位的值，因为低位全为0，经过与操作过后，低位一定是0。J 在这里是index，J 与 高位的值进行与操作过后，就能得到在扩容后面的新的index值。

设想一下，理论上我们得到的新的值应该是 `newValue = hash & (newCapacity - 1)` ，与 `oldValue = hash & (oldCapacity - 1)` 的区别仅在于高位上。 因此我们用 `J | highBit` 就可以得到新的index值。

### HashMap 是如何解决迭代问题

首先HashMap提供了3种形式的迭代方法，分别是针对Entry，Key和Value的迭代器，这里实现的代码就比较简单，看下具体的例子就可以知道。实现方式主要是依赖于对HashEntity数组进行遍历即可实现。

问题在于如何保证迭代的时候，基于正确的输出，聪明的你一定看出来了，难度在于「多线程情况的处理」。在迭代的时候，外部可以通过调用put和remove的方法，来改变正在迭代的对象。但从设计之处，HashMap自身就不是线程安全的，因此HashMap在迭代的时候使用了一种Fast—Fail的实现方式，在HashIterator里面维持了一个 expectedModCount 的变量，在每次调用的时候如果发现 `ModCount != expectedModCount`，则抛出ConcurrentModificationException 异常。但本身这种检验不能保证在发生错误的情况下，一定能抛出异常，所以我们需要在使用HashMap的时候，心里知道这是「非线程安全」的。

```java
HashMapEntry<K, V> nextEntry() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    if (nextEntry == null)
        throw new NoSuchElementException();

    HashMapEntry<K, V> entryToReturn = nextEntry;
    HashMapEntry<K, V>[] tab = table;
    HashMapEntry<K, V> next = entryToReturn.next;
    while (next == null && nextIndex < tab.length) {
        next = tab[nextIndex++];
    }
    nextEntry = next;
    return lastEntryReturned = entryToReturn;
}
```

### HashMap 是如何实现序列化接口的

HashMap是实现了Serializable接口的，这种Hash的结构如何实现序列化？按照最朴素的想法，按照默认的实现即可，但是细想之下，一个16位的数组如果只存了一个数据，却要把16位的数组到序列化进去，这本身并不可取。于是HashMap 将其他数据申明为transient，比如：

```java
transient int modCount;
```

避免这些常量参与到序列化的细节里面去，在另一方面重写 **writeObject()** 和 **readObject()**方法，通过这样的方式来实现序列化，在代码里面添加了注释，便于大家理解。

```java
private void writeObject(ObjectOutputStream stream) throws IOException {
    // Emulate loadFactor field for other implementations to read
    ObjectOutputStream.PutField fields = stream.putFields();
    fields.put("loadFactor", DEFAULT_LOAD_FACTOR);
    stream.writeFields();

    // 写入table的容量
    stream.writeInt(table.length); // Capacity
    // 写入目前的Entry数量
    stream.writeInt(size);
    for (Entry<K, V> e : entrySet()) {
    	// 迭代地写入Key和Value
        stream.writeObject(e.getKey());
        stream.writeObject(e.getValue());
    }
}
```

```java
private void readObject(ObjectInputStream stream) throws IOException,
            ClassNotFoundException {
    stream.defaultReadObject();
    int capacity = stream.readInt();
    if (capacity < 0) {
        throw new InvalidObjectException("Capacity: " + capacity);
    }
    if (capacity < MINIMUM_CAPACITY) {
        capacity = MINIMUM_CAPACITY;
    } else if (capacity > MAXIMUM_CAPACITY) {
        capacity = MAXIMUM_CAPACITY;
    } else {
        capacity = Collections.roundUpToPowerOfTwo(capacity);
    }
    makeTable(capacity);

    // 得到size大小
    int size = stream.readInt();
    if (size < 0) {
        throw new InvalidObjectException("Size: " + size);
    }

    init(); // Give subclass (LinkedHashMap) a chance to initialize itself
    for (int i = 0; i < size; i++) {
        @SuppressWarnings("unchecked") K key = (K) stream.readObject();
        @SuppressWarnings("unchecked") V val = (V) stream.readObject();
        // 构造函数里面，会计算hash放置到响应的地方
        constructorPut(key, val);
    }
}
```
