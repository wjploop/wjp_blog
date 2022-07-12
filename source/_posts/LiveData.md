### LiveData

LiveData作为一个可以观察的数据源，其目标也是实现一个消费者模式的模型，不过，它的特别之处在于其消费是具有生命周期的。



一个传统的消费者模式，就是有一个数据源,和一堆消费者，数据源维护了一个消费者集合，通过`订阅`将消费者添加到集合中，`取消订阅`就是将其从集合中删除，当数据变化时，通知集合中的消费者。

其作用是为了解耦了事件发生、和处理事件的逻辑，比如可以任意添加或删除对一个事件的处理。

传统的消费者模式在UI中通常存在一个问题，内存泄漏。消费者（Fragment, Activity）一般具有自己的生命周期，当其需要重建时，因为数据源持有它的引用，导致不能及时释放。故，一些数据层持有View层的模型，都需要手动释放View，即`取消订阅`。

而 LiveData 内部帮助我们解决了这个问题，通过将observer 绑定在 `lifecycleOwner`上，使得注册可以感应消费者的生命周期，当其内部生命周期到 `destroy`状态时便取消订阅。

```
LiveData.observe(lifecycleOwner, observer)
```

除了感应生命周期外，LiveData 也提供了一些额外的特性：

作为数据源，也想知道自己一直分发数据是否有意义？即，都没有消费者监听数据，则无意义。有一个消费者监听，则有意义。

数据源可以被 `active` 或 `inActive`，使得在没有消费者监听时，可以及时停止分发数据。



注意，对于数据关心一个不是Fragment，而Fragment中的View，因而我们一般用viewLifecycleOwner而非Fragment本身。

因为 onStart  > onResume > 跳转到其他页面 > onDestryView > 返回重新执行 onCreateView()

跳转到其他页面，Fragment不会被销毁，只是销毁其中的View，当重新创建View后，view不能马上更新到LiveData的状态。故而，我们使用 view的生命周期，在onCreatedView后，重新将LiveData的数据重新刷新到view上。