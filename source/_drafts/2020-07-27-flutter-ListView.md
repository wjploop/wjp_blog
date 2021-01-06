---
title: flutter-ListView
date: 2020-07-27 16:54:33
tags: source
categories:
---

大略浏览了 tabs 的内容后，其中的TabBar和TabBarView分别对应了安卓原生的Tab和ViewPager，很多逻辑都是一样的呀。  
其中TabBarView是PageView来实现的，看了大概之后，发觉其实它更像原生ViewPager2，而ViewPager2内部是用RecyclerView实现的，故此
我觉得应该还是先去了解Flutter中的RecyclerView，这样就找到了ListView。初略来看，ListView的文档资料的确更多一些啊。


ListView作为可以放置很多ItemView的的Widget，包括了 
* 设置滚动方向
* 设置其是否能滚动，physics
* 超出屏幕的item如何回收，如何回收，cacheExtent来设置缓存程度，(重点)
* 动态创建Item，还是一次性创建好？


itemExtent数据范围，列表的数据集大小

通常写死ListView的大小，是可以提升性能的

构建ListView有4中方法

1. 直接传入所有itemView，这样会导致所有的view都会一次性创建
2. 使用ListView.builder,使用一个IndexedWidgetBuilder，仅在需要可见时才会创建相应的itemView
3. 使用ListView.separated，使用了两个IndexedWidgetBuilder，相比于上一个，多个一个分隔符
4. 使用ListView.custom，使用 SliverChildDelegate，提供更自由的拓展。


## Child Elements 的生命周期

### 构建

当布局List，可见的Elements，States，render objects 将会创建，或复用
若是默认构造器，基于已经有的Widget来创建
若是builder，则基于已经存在的elements等来复用

### 销毁

当一个子View滚出屏幕又滚回来，同一个位置的child会使用新的elements，states,render objects

## 关于 ListView 和 CustomView
ListView 基本是一个CustomView，在其slivers属性中，赋值一个 SliverList，这样，有很多属性
都是用与CustomView的，
而CustomView.slivers属性赋值可能会是SliverList或是SliverFixedExtentList，这样便对应了
ListView是否指定了itemExtent, 即是否指定了列表的大小

```dart
ListView(
  children: <Widget>[
    Text("Hello");
  ]
);

///对应的

CustomView(
  slivers:<Widget>[
    SliverPadding(
        paddding:...
        sliver:SliverList(
          delegate: SliverChildListDelegate(
            <Widget>[
              Text("Hello")
            ]
          )     
        )
     ) 
  ]
) 
```

看完对应的CustomView之后，似乎想到了RecyclerView的嵌套问题

对于空数据的View，Flutter的设计理念？ 来个条件判断即可

## 关于 ListView的层级关系

跟android中的一样，ListView还有一个二维的GridView，但他们都继承于BoxScrollView

而，BoxedScrollView基本就是两个方法

```dart
List<Widget> buildSlivers(BuildContext context){
  buildChildLayout()
  ...
}
Widget buildChildLayout(BuildContext context);

```
没什么好看的，继续看ScrollView

## ScrollView

包含有3部分

* 一个 Scrollable Widget，监听用户手势，处理滚动
* 一个 ViewPort ，内容大小比它自身占有的大小大(我这个描述有点秀啊，原文：A widget that is bigger on the insider)
* 一个或多个 Slivers

关于这个者三部分，我们觉得重点是ViewPort。

### ViewPort

