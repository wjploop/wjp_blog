---
title: flutter_tabs
date: 2020-07-23 17:01:12
tags: source
categories:
---

```dart

import 'package:flutter/material.dart';

class TabPage extends StatefulWidget {
  @override
  _TabPageState createState() => _TabPageState();
}

class _TabPageState extends State<TabPage> with SingleTickerProviderStateMixin {
  var tabs = <Tab>[
    Tab(
      text: "动态",
    ),
    Tab(
      text: "热门",
    )
  ];
  TabController _tabController;

  @override
  void initState() {
    super.initState();
    _tabController = TabController(length: tabs.length, vsync: this);
  }

  @override
  void dispose() {
    _tabController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
          bottom: TabBar(
            tabs: tabs,
            controller: _tabController,
          ),
      ),
      body: TabBarView(children: tabs.map((e) =>
          Center(
            child: Text(
              "Hey,I am content of $e", style: TextStyle(fontSize: 18),),
          )).toList()),
    );
  }
}

void main() {
 
  runApp(TabPage());

}
```

## Scaffold 的创建需要传入带有 MediaQuery 数据的 Context

报错如下

```
I/flutter ( 2928): The following assertion was thrown building TabPage(state: _TabPageState#f2fb0(ticker inactive)):
I/flutter ( 2928): MediaQuery.of() called with a context that does not contain a MediaQuery.
I/flutter ( 2928): No MediaQuery ancestor could be found starting from the context that was passed to MediaQuery.of().
I/flutter ( 2928): This can happen because you do not have a WidgetsApp or MaterialApp widget (those widgets introduce
I/flutter ( 2928): a MediaQuery), or it can happen if the context you use comes from a widget above those widgets.
I/flutter ( 2928): The context used was:
I/flutter ( 2928):   Scaffold
```

Scaffold作为一个脚手架，创建时不能直接作为根视图，还是需要外包 MaterialApp 等，其创建后，可以为子Widget提供一些数据，比如当前
屏幕的大小，像素密度等。

我们看看 WidgetsApp 如何为下层提供MediaQuery相关数据的？

我们看到其内部使用一个MediaQueryFromWindow包裹，即从Window中获取屏幕的信息，其获取方法使用使用 WidgetsBindingObserver 的方式，
而在MediaQuery中，使用静态方法，将data绑定到Context，后续子Widget想要获取该信息就可以直接通过下面获取
```
 var data = MediaQuery.of(context)
```
另外，由于MediaQuery是继承 InheritedWidget的，当子Widget实现了didChangeDependencies方法后，当屏幕信息改变，根视图WidgetsApp会
通过观察者WidgetsBindingObserver观察到，重新设置当前的MediaQueryData，之后子Widget也能收到屏幕变化，从而重建。
```dart
/// Builds [MediaQuery] from `window` by listening to [WidgetsBinding].
///
/// It is performed in a standalone widget to rebuild **only** [MediaQuery] and
/// its dependents when `window` changes, instead of rebuilding the entire widget tree.
class _MediaQueryFromWindow extends StatefulWidget {
  
}
class _MediaQueryFromWindowsState extends State<_MediaQueryFromWindow> with WidgetsBindingObserver {
    @override
    Widget build(BuildContext context) {
      // 通过Bindg实例的window，连通我们的屏幕，获取相关的信息
      MediaQueryData data = MediaQueryData.fromWindow(WidgetsBinding.instance.window);
      if (!kReleaseMode) {
        data = data.copyWith(platformBrightness: debugBrightnessOverride);
      }
      return MediaQuery(
        data: data,
        child: widget.child,
      );
    }
}
```

```dart
 /// The data from the closest instance of this class that encloses the given
  /// context.
  ///
  /// You can use this function to query the size an orientation of the screen.
  /// When that information changes, your widget will be scheduled to be
  /// rebuilt, keeping your widget up-to-date.
  ///
  /// Typical usage is as follows:
  ///
  /// ```dart
  /// MediaQueryData media = MediaQuery.of(context);
  /// ```
  ///
  /// If there is no [MediaQuery] in scope, then this will throw an exception.
  /// To return null if there is no [MediaQuery], then pass `nullOk: true`.
  ///
  /// If you use this from a widget (e.g. in its build function), consider
  /// calling [debugCheckHasMediaQuery].
  static MediaQueryData of(BuildContext context, { bool nullOk = false }) {
    assert(context != null);
    assert(nullOk != null);
    final MediaQuery query = context.dependOnInheritedWidgetOfExactType<MediaQuery>();
    if (query != null)
      return query.data;
    if (nullOk)
      return null;
    throw FlutterError.fromParts(<DiagnosticsNode>[
      ErrorSummary('MediaQuery.of() called with a context that does not contain a MediaQuery.'),
      ErrorDescription(
        'No MediaQuery ancestor could be found starting from the context that was passed '
        'to MediaQuery.of(). This can happen because you do not have a WidgetsApp or '
        'MaterialApp widget (those widgets introduce a MediaQuery), or it can happen '
        'if the context you use comes from a widget above those widgets.'
      ),
      context.describeElement('The context used was')
    ]);
  }
```

### WidgetsApp 与 MaterialApp 的区别


看看前者WidgetsApp的定义,打包了一个应用普遍需要的Widget
>/// A convenience widget that wraps a number of widgets that are commonly  
 /// required for an application.

而 MaterialApp 可以说是在前者的基础上实现了Material风格，与之对应的风格还有苹果风格 CupertinoApp

> /// A convenience widget that wraps a number of widgets that are commonly
  /// required for material design applications. It builds upon a [WidgetsApp] by
  /// adding material-design specific functionality, such as [AnimatedTheme] and
  /// [GridPaper].


### 好了，我们还是回到 TabController 吧

看看定义，就是协同 `Tabbar` 和 `TabBarView` 选中关系的，两者都有自己的index，当其中一个index发生改变时，另一个index也要跟着
变化哈。

TabController作为两者其中的沟通的桥梁

其中有个比较关键的点，其中的Animation中的值, 值的范围为[0-tabs.length],是double类型，代表着TabBar 和 TabBarView的滚动偏移量
scrollOffset


```dart
/// 
/// Coordinates tab selection between a [TabBar] and a [TabBarView].
///
/// The [index] property is the index of the selected tab and the [animation]
/// represents the current scroll positions of the tab bar and the tab bar view.
/// The selected tab's index can be changed with [animateTo].
///
/// A stateful widget that builds a [TabBar] or a [TabBarView] can create
/// a [TabController] and share it directly.
///
/// When the [TabBar] and [TabBarView] don't have a convenient stateful
/// ancestor, a [TabController] can be shared by providing a
/// [DefaultTabController] inherited widget.
///
  
 
  /// An animation whose value represents the current position of the [TabBar]'s
  /// selected tab indicator as well as the scrollOffsets of the [TabBar]
  /// and [TabBarView].
  ///
  /// The animation's value ranges from 0.0 to [length] - 1.0. After the
  /// selected tab is changed, the animation's value equals [index]. The
  /// animation's value can be [offset] by +/- 1.0 to reflect [TabBarView]
  /// drag scrolling.
  ///
  /// If this [TabController] was disposed, then return null.
  Animation<double> get animation => _animationController?.view;
  AnimationController _animationController;

  /// 易见的思维，维护两个值
  int _previousIndex;
  int _index;
  
  /// 可以看到改变index时，会执行相应的动画哈
  void _changeIndex(int value, { Duration duration, Curve curve }) {
    if (value == _index || length < 2)
      return;
    _previousIndex = index;
    _index = value;
    if (duration != null) {
      _indexIsChangingCount += 1;
      notifyListeners(); // Because the value of indexIsChanging may have changed.
      // 使用AnimationController来执行动画
      _animationController
        .animateTo(_index.toDouble(), duration: duration, curve: curve)
        .whenCompleteOrCancel(() {
          _indexIsChangingCount -= 1;
          notifyListeners();
        });
    } else {
      _indexIsChangingCount += 1;
      _animationController.value = _index.toDouble();
      _indexIsChangingCount -= 1;
      notifyListeners();
    }
  }
```

在TabBar中我们就可以看大调用controller方法的例子

```dart

    /// 这里我们及监听frame的变化哦，这样，几乎实时更TabPagView保持同步偏移
  void _handleTabControllerAnimationTick() {
    assert(mounted);
    if (!_controller.indexIsChanging && widget.isScrollable) {
      // Sync the TabBar's scroll position with the TabBarView's PageView.
      _currentIndex = _controller.index;
      _scrollToControllerValue();
    }
  }

  
  void _handleTabControllerTick() {
    if (_controller.index != _currentIndex) {
      _currentIndex = _controller.index;
      if (widget.isScrollable)
        _scrollToCurrentIndex();
    }
    setState(() {
      // Rebuild the tabs after a (potentially animated) index change
      // has completed.
    });
  }
  /// 最典型的例子，点击tab，导致controller对应的tabPageView也会滚动到对应页面
  void _handleTap(int index) {
    assert(index >= 0 && index < widget.tabs.length);
    _controller.animateTo(index);
    if (widget.onTap != null) {
      widget.onTap(index);
    }
  }
```


### 看看 _TabBarState的实现

```dart

class _TabBarState extends State<TabBar> {
  // Tabs 一般可以左右滑动，用于控制当前TabBar的整体偏移量
  ScrollController _scrollController;
  // 与TabBarView交换数据的
  TabController _controller;
  // 指示器，这个时候很有搞头，可以做一些炫酷（花哨）的效果 todo 了解在画布上自己画东西
  _IndicatorPainter _indicatorPainter;
  int _currentIndex;
  // 基本指示器的宽度是如何确定的呢
  double _tabStripWidth;
  // GlobalKey最大的作用，就是可以reparent，可以在Element树中更换位置， todo 了解这些Key的作用
  List<GlobalKey> _tabKeys;
  
  @override
  Widget build(BuildContext context) {
    
    /// 似乎也挺简单的，所有天的tab添加点击事件
    // Add the tap handler to each tab. If the tab bar is not scrollable,
    // then give all of the tabs equal flexibility so that they each occupy
    // the same share of the tab bar's overall width.
    final int tabCount = widget.tabs.length;
    for (int index = 0; index < tabCount; index += 1) {
      // 包裹InkWell 墨水池，挺形象的，水波效果
      wrappedTabs[index] = InkWell(
        mouseCursor: widget.mouseCursor ?? SystemMouseCursors.click,
        onTap: () { _handleTap(index); },
        child: Padding(
          padding: EdgeInsets.only(bottom: widget.indicatorWeight),
          child: Stack(
            children: <Widget>[
              wrappedTabs[index],
              // 这个为视觉不好的人，辅助功能呀
              Semantics(
                selected: index == _currentIndex,
                label: localizations.tabLabel(tabIndex: index + 1, tabCount: tabCount),
              ),
            ],
          ),
        ),
      );
    // 若是不可滚动，这是使用Expanded包裹，所有的tab能平分水平距离
      if (!widget.isScrollable)
        wrappedTabs[index] = Expanded(child: wrappedTabs[index]);
    }

    Widget tabBar = CustomPaint(
      painter: _indicatorPainter,
      child: _TabStyle(
        animation: kAlwaysDismissedAnimation,
        selected: false,
        labelColor: widget.labelColor,
        unselectedLabelColor: widget.unselectedLabelColor,
        labelStyle: widget.labelStyle,
        unselectedLabelStyle: widget.unselectedLabelStyle,
        child: _TabLabelBar(
          // 重新布局时，保存当前tab的Offset，内部TabLabelBar的实现
          onPerformLayout: _saveTabOffsets,
          children: wrappedTabs,
        ),
      ),
    );

    if (widget.isScrollable) {
      _scrollController ??= _TabBarScrollController(this);
      // 为Android的ScrollView的取了一个更精确的名字
      tabBar = SingleChildScrollView(
        dragStartBehavior: widget.dragStartBehavior,
        scrollDirection: Axis.horizontal,
        controller: _scrollController,
        physics: widget.physics,
        child: tabBar,
      );
    }

    return tabBar;
  }
}


class _TabBarViewState extends State<TabBarView> {
  TabController _controller;
  PageController _pageController;
  // 这里我们看到children不是直接用了，而是附加上一Key，这个Key是根绝Index来生成了
  //_childrenWithKey = KeyedSubtree.ensureUniqueKeysForList(widget.children);
  // 为什么呢？
  // Creates a KeyedSubtree for child with a key that's based on the child's existing key or childIndex.
  // Key和对应的index是确定的，似乎就是保存当前视图的状态了
  List<Widget> _children;
  List<Widget> _childrenWithKey;
  int _currentIndex;
  int _warpUnderwayCount = 0;


  @override
  Widget build(BuildContext context) {
    // 这里使用NotificationListener,就是要要监听冒泡上来的事件，传入的泛型就是它想关注的通知
    return NotificationListener<ScrollNotification>(
      onNotification: _handleScrollNotification,
      child: PageView(
        dragStartBehavior: widget.dragStartBehavior,
        controller: _pageController,
        physics: widget.physics == null
          ? _kTabBarViewPhysics.applyTo(const ClampingScrollPhysics())
          : _kTabBarViewPhysics.applyTo(widget.physics),
        // 
        children: _childrenWithKey,
      ),
    );
  }
}

```
