---
title: dart异步编程：Isolate和事件循环（译）
date: 2021-09-10 15:21:53
tags: code flutter
categories:
---

> [原文](https://medium.com/dartlang/dart-asynchronous-programming-isolates-and-event-loops-bffc3e296a6a)

Dart, 尽管作为一个单线程语言，但也提供了一系列的API，诸如Future, Stream，来帮助我们编写一个现代、异步、响应式的程序。本文主要讲述Dart在底层如何支持这样的后台任务，包括 isolate, 事件循环。

## Isolate

Isolate，指代码运行之处，指Dart程序所拥有的内存空间，以及在其之上运行的事件循环。

在一些类似C++的语言中，支持多个线程共享一段内存空间，每个线程可以运行你想要的代码块（堆、方法区共享）。而在dart中，一个线程就是一个isolate和其持有的内存区，其线程就只是一直在处理事件。

通常一个Dart程序只会开启一个Isolate, 但也支持开启多个。若是有一个耗时任务在主线程中执行，会导致app丢帧，就可以用 `Isolsate.spawn()`, `Flutter的 compute()`方法，两者都会开启一个新的Isolate， 使得主Isolate忙于重建、渲染widget。

新的Isolate拥有自己的事件循环和内存空间，即使是由父类孵化(Spawn)出来的，父级也不允许访问孵化出来的Isolate。也就如其名所言，Isolate岛屿之意。

实际上，Isolate之间的交互只能通过相互发送信息。，相比于Java/C++不同线程共享一个内存空间的机制而言，这种缺乏共享内存的设计可能看起来过于严格，效率不高。但Dart定位为客户端语言来说，却也有一定好处。比如，内存分配，垃圾回收不必需要使用锁了。因为只有一个线程，对于一个段内存空间而言，在某一个时刻，永远只会有一个CPU在访问。这样的特点，非常适用于客户端程序，因为需要频繁的重建、销毁原来的widget对象。

## Event Loops 事件循环

以上简单介绍了Isolate，让我们继续探究一个单线程如何实现异步的，即事件循环。

想象一个app的生命周期，包括了开始、结束、和中间一些系列事件的发生，包括io事件、用户点击事件等。

你的程序并不能预测这些事件何时会发生，所以你只能开始时定义好收到什么事件时，该如何处理。

>> 不想继续翻译了，后面重复一个概念，告诉我们，所有无论是启动一个Callback、或开启一个Future，都是在添加一个事件处理到事件循环中，当其被调用时，只是收到了一个事件。



## 本人后续思考补充



### 关于dart单线程的设计在客户端中具有优势？

感觉优势不是很大呀。即便在支持多线程的语言的Android中，管理UI会由唯一的UI线程来处理。即便是对于SurfaceView的场景来说，有了额外的线程来绘制UI，在展示时还是得交由UI线程保证每一帧的有序。但，这应该不是dart所提到相对优势的点，多个线程同时处理一个UI的框架，似乎没有遇到过。

### dart有俩个队列，micro, event，为啥Android只有一个队列呢？

问答这个问题，得先了解dart为什么要分了俩个队列，其作用是什么？


![flowchart: main() -> microtasks -> next event -> microtasks -> ...](https://dart.cn/articles/archive/images/both-queues.png)

通过这个图，可知，microtask的优先级很高，每次从event队列取出一task时，会先把micro队列的task执行完。

假设我们 new 俩个future，相当于把俩个task A, B 依次放入到 event队列中，这里，我们可以保证 A > B的顺序。

在执行A时，我们又 new 了 task C, 那么，可以保证顺序 A > B > C。

若是在执行 task A时，需要执行一个比较急迫的任务 urgentC，需要保证 A > ugentC > B,  故需要将urgentC放入到micro队列。

为什么会有这么一个需求呢？因为我们不知道后续的B需要执行多久，或是A，B同时修改一个状态时，我们的 urgentC需要依赖A修改后的状态。或是，我们在执行A时，发现不需要后续的B执行了。这个有点像Android Handler的removeMessage，比如急着将对象引用断开。可以注意到的是，这个急迫的 urgentC的队列为啥叫 microtask queue, 相比于急迫，更重要的是要保证该任务是轻量的。 实际上很多task放入event队列，比如响应用户的事件。

简单说，microd task就是保证该任务会优先于后续的event task执行。既然存在这么一个需求，dart存在micro task方案，那么在Android中肯定也有对应的方案。是的，MessageQueue存在一个同步屏障的概念，sync barrier。postSyncBarrier() 有这么一段注释:

> ```
>      * This method is used to immediately postpone execution of all subsequently posted
>      * synchronous messages until a condition is met that releases the barrier.
>      * Asynchronous messages (see {@link Message#isAsynchronous} are exempt from the barrier
>      * and continue to be processed as usual.
> 
>       该方法用于推迟处理后续提交同步消息，直至同步屏障的释放。而异步消息则免于其干扰正常执行。
> ```

在Android中急迫的task是根据 异步任务和同步屏障配合而生效的，入队列的任务，当 barrier出现时，先优先执行异步消息。

异步任务与dart的micro task有一点区别，dart一旦发布micro task，必将优先一个个执行完。而android的异步任务，其优先性的是依赖barrier而生效的，可以说异步任务更为灵活点。

