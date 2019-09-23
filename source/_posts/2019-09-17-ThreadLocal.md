---
title: ThreadLocal
date: 2019-09-17 15:52:51
tags: java 
categories:
- code 
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
即，不使用这个类，所有变量都是指向堆中的对象，只要一个线程修改了，其他线程也会受到影响。  
的确，是会受到影响的，只不过以为每个线程都有自己的缓存的原因，它们所所谓的共有，不能做到及时
同步，需要额外处理，否则一般会出现bug的。 这里，可以使用volatile关键字强制同步。参见`单例模式一文`


而这里的线程私有，是指该对象只能从该线程获取，比如线程A创建的对象obj,B对象试图获取，得到的是null。 


看看源码，其实现还是很简单的，  

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

> 好吧，这个ThreadLocalMap中的弱引用没搞懂啊~


其中，他的GET，SET方法，都是先通过Thread.currentThread来获取到当前调用方法的线程，在该线程的ThreadLocalMap中检查好是否设置有该变量。

