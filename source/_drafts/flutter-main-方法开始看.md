---
title: flutter_main_方法开始看
tags:
---



```dart
/// Inflate the given widget and attach it to the screen.
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..scheduleAttachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```

启动一个Flutter项目，就是将传入Widget"吹"起来，并渲染到屏幕中。   
 
这么说有点莫名其妙，首先了解到Widget的定义。

要了解Widget，可以对比Android中的View来理解。View中包含了两部分信息。 

1. 用户界面是怎么样的；如，当前视图的颜色，大小。
2. 页面如何绘制具体过程；如View是否已经加载显示的状态信息。

而在Flutter中，`Widget`仅包前者，后者是对应的是叫做渲染树Render Tree，节点是`Element`。故，可以说，android中
的`View`包含了`Widget`和`Element`两者的信息，在设计上，flutter耦合度更低了。Flutter这么处理是有好处的。

1. 我们在`画`页面时，只要关注如何如何画就好了，关注的信息更加少。
2. 使用Widget之后，每一次状态的变化，我们都会创建创建一个新Widget，新旧Widget的变化，交给Flutter框架来对比，计算出最小
的变化后，将变化应用到旧的UI上。这样带来的思考方式的改变，我们不用思考如何将A转换成B的具体操作，而是描述变化后B的样子，
然后由框架来帮忙处理力。



