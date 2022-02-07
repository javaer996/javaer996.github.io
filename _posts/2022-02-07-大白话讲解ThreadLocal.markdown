---
layout: post
read_time: true
show_date: true
title:  大白话讲解ThreadLocal
date:   2022-02-07 13:35:20 +0800
description: threadlocal ThreadLocal
img: posts/common/java.png
tags: [java,threadlocal]
author: tengjiang
toc: yes
---

|| 该文章主要介绍ThreadLocal原理以及使用时需要注意的问题。||

<!-- more -->



## 什么是ThreadLocal?

ThreadLocal顾名思义，本地线程，可以理解为线程局部变量，说白了就是操作本地线程的局部变量，通过使用ThreadLocal可以为每一个线程持有一个副本，每个线程中的变量的值互不影响。

## 数据存在哪里？

存在Thread类中的threadLocals字段中，也就是每个线程的局部变量是存在于自己的线程中的，该字段类型为ThreadLocal.ThreadLocalMap，即为一个Map。

> 注意：因为数据是存储在Thread对象中的，所以如果使用线程池，线程使用完后没被销毁，并且没有手动清除该ThreadLocal的内容，那么在其他功能使用到该线程的时候，是可以获取到上次没被清除的ThreadLocal内容的。

## 如何使用？

```java
public class Main {

    // ThreadLocal是接收泛型的，所以我们可以通过ThreadLocal存储任意信息。这里我们存储字符串String
    // 定义ThreadLocal对象，通过它可以操作Thread类中的threadlocals字段
    private static ThreadLocal<String> threadLocal = new ThreadLocal();

    public static void main(String[] args) {
      try{
        // 存值，实际上是将该字符串存储到该线程对象Thread对应的threadlocals字段的Map对象中
        // map中的key为该threadLocal对象
        threadLocal.set("thread local demo");
        // 取值，实际上是从该线程对象Thread对应的threadLocals字段中取key为threadLocal对象的值
        String value = threadLocal.get();
        System.out.println(value);
      }finally {
        // 手动清除threadLocal内容，防止内存泄露
        threadLocal.remove();
      }
    }
}
```

> ThreadLocal对象不存储值，它仅仅是作为一个Key，通过它实际操作的是该线程Thread对象的threadLocal字段

## 内存泄露是怎么回事？

ThreadLocal存在内存泄露的根本原因是：**没有在使用完之后调用remove()方法清除数据**。

想要出现内存泄露需要满足以下四个条件：

- 使用完后没有调用remove方法清除数据；
- 使用完后再也没有通过该threadLocal调用过get()/set()方法；
- ThreadLocal对象已不再使用，并且被回收；
- Thread线程对象一直存活；

## ThreadLocal源码解析

### threadLocals

```java
public class Thread {
  ...
    // 通过ThreadLocal实际操作的就是该字段
    ThreadLocal.ThreadLocalMap threadLocals = null;
  ...
}
```

###  ThreadLocal

```java
public class ThreadLocal {
  
  // 取值
  public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
      // 从ThreadLocalMap取值，key即threadLocal对象
      ThreadLocalMap.Entry e = map.getEntry(this);
      if (e != null) {
        T result = (T)e.value;
        return result;
      }
    }
    // 设置初始值，实际上没有设置任何值，直接返回了null,但是这里会创建ThreadLocalMap
    return setInitialValue();
  }

  // 针对当前ThreadLocal设置初始值
  private T setInitialValue() {
    // 默认直接返回null，没有处理，子类可以实现该方式进行实际初始化逻辑
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
      map.set(this, value);
    else
      // 创建ThreadLocalMap
      createMap(t, value);
    return value;
  }
  
    // 设置值
  public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
      map.set(this, value);
    else
      // 创建ThreadLocalMap
      createMap(t, value);
  }
  
  // 获取当前线程的threadLocals字段的ThreadLocalMap对象
  ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
  }
  
  // 初始化Thread的threadLocals字段，创建ThreadLocalMap对象
  void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
  }

  // 移除值，防止内存泄露就是靠它
  public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
      m.remove(this);
  }
  
  static class ThreadLocalMap {

    // 可以看出ThreadLocalMap中存储的key泛型为ThreadLocal，并且是弱引用类型，也就是如果进行垃圾回收的话，
    // 当ThreadLocal已被回收不再使用的时候，该key会变成null，也就是此时key为null，value仍然是之前存储的内容
    static class Entry extends WeakReference<ThreadLocal<?>> {
      Object value;
      Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
      }
    }

    private static final int INITIAL_CAPACITY = 16;
    // 实际存储内容的数组,类型为上面的Entry实体
    private Entry[] table;
    ...
    
    // 实际获取值的地方
    private Entry getEntry(ThreadLocal<?> key) {
      int i = key.threadLocalHashCode & (table.length - 1);
      Entry e = table[i];
      if (e != null && e.get() == key)
        return e;
      else
        // 如果没有获取的话，会处理内存泄露的问题
        return getEntryAfterMiss(key, i, e);
    }

    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
      Entry[] tab = table;
      int len = tab.length;

      while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
          return e;
        if (k == null)
          // 如果value存在，key为null，则会清除value内容，防止内存泄露
          expungeStaleEntry(i);
        else
          i = nextIndex(i, len);
        e = tab[i];
      }
      return null;
    }
    
    // 实际设置值的地方
    private void set(ThreadLocal<?> key, Object value) {
      Entry[] tab = table;
      int len = tab.length;
      int i = key.threadLocalHashCode & (len-1);

      for (Entry e = tab[i];
           e != null;
           e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
          e.value = value;
          return;
        }

        if (k == null) {
          // 如果值存在，key为null，会用新的entry替换旧的
          replaceStaleEntry(key, value, i);
          return;
        }
      }

      tab[i] = new Entry(key, value);
      int sz = ++size;
      if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
    }

    // 实际清理值的地方
    private void remove(ThreadLocal<?> key) {
      Entry[] tab = table;
      int len = tab.length;
      int i = key.threadLocalHashCode & (len-1);
      for (Entry e = tab[i];
           e != null;
           e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
          e.clear();
          expungeStaleEntry(i);
          return;
        }
      }
    }
	}
}
```

总结：

- ThreadLocalMap中的key为弱引用，如果ThreadLocal已被回收不再使用，ThreadLocalMap中的Key对应的threadLocal引用也会被回收，key变为null，但是value并不会被回收。
- ThreadLocal的使用get()/set()方法操作时，都会对value值存在，key为null的情况进行处理。
- 如果不想出现内存泄露，正确的使用姿势就是在使用完成后手动调用remove()方法进行清除。

## 扩展知识

> Java的强引用，软引用，弱引用，虚引用：

- 强引用：**强引用**是使用最普遍的引用。如果一个对象具有强引用，那**垃圾回收器**绝不会回收它。
- 软引用：如果一个对象只具有**软引用**，则**内存空间充足**时，**垃圾回收器**就**不会**回收它；如果**内存空间不足**了，就会**回收**这些对象的内存。
- 弱引用：在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有**弱引用**的对象，不管当前**内存空间足够与否**，都会**回收**它的内存。
- 虚引用：顾名思义，就是**形同虚设**。与其他几种引用都不同，**虚引用**并**不会**决定对象的**生命周期**。如果一个对象**仅持有虚引用**，那么它就和**没有任何引用**一样，在任何时候都可能被垃圾回收器回收。**虚引用**主要用来**跟踪对象**被垃圾回收器**回收**的活动。


