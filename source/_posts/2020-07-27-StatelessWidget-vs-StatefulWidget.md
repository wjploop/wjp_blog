---
title: StatelessWidget_vs_StatefulWidget
date: 2020-07-27 18:01:17
tags:
categories:
---

这两个类在framework.dart中，可见其重要性。其实自己看过了多次，不过现在还是想用文字记录着再看一次，因为它们太重要了。

## 两者还有一个父类，Widget

```dart

/// Element的配置信息
abstract class Widget {
  final Key key;
  // Widget的存在的目的就是创建出Element，作为元素再Element树中
  // 其创建过程我们称之为inflate，注意可以加载多次，对应多个Element
  Element createElement();
  
  // 当 oldWidget 创建的Element加载在树中后
  // 当我们再次调用build方法时，对于原先的Element是怎么处理呢？
  // 我们是否能够更新原先老的Element？
  // 这个问题很重要，决定了Element能否被复用，而不是重新创建的新的Element来替换掉原先的Element。
  // 这个问题的换个说法，就是原先的Element能否被更新？ 
  // 答案是，需要runtime和key都相同。
  // 注意，当key都为空时，也认为key相同，当然这个很符合逻辑。
  bool canUpdate(Widget oldWidget, Widget newWidget){
    // runtimeType && key equal
  }
}
```


## StatelessWidget

```dart

/// 强调了UI仅依赖配置信息，和当前环境BuildContext
///
/// 性能考虑
/// 
/// build()方法的执行时机有三种
/// 1. 第一次插入到 Element 树中
/// 2, 父级的Widget改变，递归到导致该方法执行
/// 3, 依赖信息改变，其依赖的信息形式是InheritedWidget
/// 
/// 尽管build方法的执行，并不会每次都会重新新的Element，但还是避免频繁调用build方法的。措施如下下
/// 减少节点层级，使得节点递归创建。如Align替代 Row，Column 的混用
/// 使用const修饰构造器，可以避免每次setState而重建
/// 考虑使用statefulWidget来利用相关的缓存技术
/// 在使用InheritedWidget时，将依赖更加精细化
abstract class StatelessWidget{
  
  StatelessElement createElement() => StatelessElement(this);
  
  Widget build(BuildContext context);

}
```

## StatefulWidget

状态信息可以异步读取  
状态信息可以在Widget的生命周期内修改

注意到，StatefulWidget本身是不可变，只有State是可变的，其属性State是final field，创建后后指向。

每次 inflate Widget 便会调用createSate()。从树中移除后，重新插入也会导致创建createState

当使用GlobalKey时，移动Element的位置，可以复用原来的State，这样我们可以移动一个子树，注意移除还插入的动作需要在动画的同一帧完成。


性能考虑：

将State尽可能放到叶子，下放   
要是子树不会改变，缓存子树  
避免修改子树的深度和类型，转而使用属性来控制相应的状态。这个可以用 canUpdate() 来理解，改变状态来可以避免原先Element被替换。
若是子树深度非要改变，子树使用GlobalKey。使用KeyedSubtree要是不方便指定Widget来使用GlobalKey。

### State

Widget创建的State绑定到 BuildContext 被认为是 mounted状态，其关联关系绑定后不可变了。

绑定后，可以根据Context来 initSate()



```dart
abstract class StatefulWidget{
  
  
  StatefulWidget createElement() => StatefulElement(this);
  
  State createState();
}

enum _StateLifeCycle{
  // initState()未执行
  created,
  // 需要执行didChangeDependencies（）
  initialized,
  // 可以执行build，且dispose未执行
  ready,
  // dispose()已执行，不能再重新build
  defunct
}

abstract class State<T extends StatefulWidget> {
  
  T _widget;
  
  // widget z
  BuildContext get context => _element;
  StatefulElement _element;
  
  bool get mounted => element != null;
  
  // 当Widget插入到树中时调用
  
  void initState(){
  }
  
  // 当配置发生变化调用
  // 当父级Widget重建需要更新该Widget，（runtimeType和Key相同），导致的结果是
  // 更新该State，基于原先的oldWidget创建新的Widget
  
  //  在这个方法后，回到调用build方法的，所以无需再手动setState

  void didUpdateWidget(covariant T oldWidget){}
  
  // 通知框架，该State中的属性改变了，需要更新
  // 
  /// Future<void> _incrementCounter() async {
  ///   setState(() {
        // 影响Widget的放在里面
  ///     _counter++;
  ///   });
        // 其他不直接影响的放外面
  ///   Directory directory = await getApplicationDocumentsDirectory();
  ///   final String dirName = directory.path;
  ///   await File('$dir/counter.txt').writeAsString('$_counter');
  /// }
  void setState(VoidCallback fn){
    final dynamic result = fn() as dynamic;
    _element.markNeedBuild();
  }
  
  // 当State在被移除树时
  // 当State更新，从一个位置移动另一个位置时，在该方法调用后，会保证调用build方法的
  // 注意哦
  // 由于移除和插入是两个步骤，且插入的动作需要保证在动画的新一帧绘制前完成，故需要该方法执行要快,故释放资源的操作该
  // 放在 dispose（）中
  void deactivate(){}

  void dispose();
  
  // 该方法可能在下列情况下执行
  // after initState
  // after didUpdateWidget
  // after receive a call to setState
  // a dependency of State change
  // after call deactivate and reinsert State into other location
  //
  // 

  Widget build(BuildContext context);

  void didChangeDependencies() { }
  
}
```

### 一个设计问题？为什么 Build（）放在State，而不是放在StatefulWidget

没有在StatefulWidget中这么写，给StatefulWidget更多的灵活性，如何理解呢？
```dart
Widget build(BuildContext context, State state) {}
```

AnimateWidget是很多动画的表示，假设在StatefulWidget中防止Build方法，则导致Widget依赖很多动画的实现State

概念上，StatelessWidget是StatefulWidget的子类，若是将StatefulWidget多了个方法，那么将不可能了


### State中的注册和注销

假设我们构造的目标对象，依赖了某些对象，我们该在什么时机注册和注销这些依赖呢

initState() 中订阅

didUpdateWidget() 中，若是需要，unSubscribe原先的源并更换新的源

dispose() 注销所有的源


