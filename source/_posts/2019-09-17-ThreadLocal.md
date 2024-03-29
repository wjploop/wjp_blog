---
title: ThreadLocal
date: 2019-09-17 15:52:51
tags: java 

---

## ThreadLocal

---


/**
 * This class provides thread-local variables.  These variables differ from
 * their normal counterparts in that each thread that accesses one (via its
 * {@code get} or {@code set} method) has its own, independently initialized
 * copy of the variable.  {@code ThreadLocal} instances are typically private
 * static fields in classes that wish to associate state with a thread (e.g.,
 * a user ID or Transaction ID).
 *

 

官方解释，即一个提供线程私有变量。   
既然这里说，使用这个类可以提供一个线程私有的变量，那么就是说，不使用，就是线程共有的了。  
即，不使用这个类，所有变量都是指向堆中的同一个个对象，只要一个线程修改了，其他线程也会受到影响。  

>的确，是会受到影响的，只不过以为每个线程都有自己的缓存的原因，它们所谓的共有，不能做到及时
同步，需要额外处理，否则一般会出现bug的。 这里，可以使用volatile关键字强制同步。参见`单例模式一文`


而这里的线程私有，是指该对象只能从该线程获取，比如线程A创建的对象obj,B对象试图获取，得到的是null。 


看看源码，其实现还是相对简单的，  

在Thread内持有一个ThreadLocalMap,里面保存有该线程所有线程私有变量。  

其中，为了避免内存泄漏

```java
static class Entry extends WeakReference<ThreadLocal> {
    /** The value associated with this ThreadLocal. */
    Object value;
    Entry(ThreadLocal k, Object v) {
        super(k);
        value = v;
    }
}
```

ThreadLocal有个内部类ThreadLocalMap, 使用了一个很简易的HashMap实现方式,  
不同的是，当有冲突时，其没有使用链表来保存额外的Entry,而是方法table的下一位，即线性检测。  
原因是，应该是认为一个线程不应该有那么多私有变量啊。 

两个点是，使用WeakReference来包裹Entry,这样，要回收把一个变量放到ThreadLocal中，并不会持有该对象的强应用，若是只有
该放到ThreadLocal这个对象，ThreadLocal<Object> 整体作为Key，而Value是Object，当然Key为空null，table中也只是保存有entry，
所以当只有entry指向该value时,  
```java

ThreadLcoal<Object> obj = new Object();

obj作为一个key在Thread的LocalThreadMap中，若是key为null，需要该指向的value也为
```
弱引用,
```java
WeakReference<A> aRef = new WeakReference<A>();
A a = new A();
aRef.set(a);
//持有有A对象有两个应用，一个强一个弱
//此时
System.gc();
//实例A不会回收

a=null
再次
System.gc();
仅剩弱引用指向，故，实例A会被回收

```
而在ThreadLcoalMap中，保存的实例
```java
var t = new ThreadLocal<String>()
//将t作为key放入当前线程中的map中，
var data = "hello"

t.set(data)
//当设值之后，data这个对象有两个引用，
data=null   //去掉一个

还有一个，在当前线程还活着的状态
current_thread.threaLocalMap.entry.value
//map中的tables存放弱引用，故get()时，随时都可能==null，
故，我觉得该data对象会被回收...
但是，网上的文章说不会，没搞明白
t.remove()
```

> 好吧，这个ThreadLocalMap中的弱引用没搞懂啊~


其中，他的GET，SET方法，都是先通过Thread.currentThread来获取到当前调用方法的线程，在该线程的ThreadLocalMap中检查好是否设置有该变量。

