---
title: Handler中的AndroidLeak
date: 2019-09-17 15:52:24
tags: android

---


>Since this Handler is declared as an inner class, it may prevent the outer class from being garbage collected. If the Handler is using a Looper or MessageQueue for a thread other than the main thread, then there is no issue. If the Handler is using the Looper or MessageQueue of the main thread, you need to fix your Handler declaration, as follows: Declare the Handler as a static class; In the outer class, instantiate a WeakReference to the outer class and pass this object to your Handler when you instantiate the Handler; Make all references to members of the outer class using the WeakReference object.

Handler不恰当的使用方式，会导致内存泄漏。  

若是在Activity中使用非静态内部类的Handler时，该Handler会持有外部Activity的引用，（内部类的特点）。  
使用handler发送出去的消息Message，其内部会持有一个handler的引用，即msg->handler->Activity.  d
在Activity结束时，msg可能还在在消息队列中等待处理，所以，会导致有Activity对象不能得到回收。

针对这个问题，最简单的方法是，Activity的生命周期方法的onDestroy方法中调用removeAllCallBack(null),其方法会导致所以待处理的Message
作废，即所以target指向为该handler消息都移除队列，并且target指向null。

另外一种方法是，使用Handler使用静态类，即其类的创建不再有指向Activity的引用了。但是一般情况，我们handler的处理消息，都要使用到Activity
来处理啊，因而，我们在该静态的Handler类中，使用WeakReference来包装该Activity，即使用这个`有实无名`的Activity， （嗯~，渣男）