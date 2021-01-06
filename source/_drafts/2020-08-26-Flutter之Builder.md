---
title: Flutter之Builder
date: 2020-08-26 11:32:55
tags: flutter
categories:
---

例子

```dart

class App extends StatefulWidget {
  @override
  _AppState createState() => _AppState();
}

class _AppState extends State<App> {
  @override
  Widget build(BuildContext context) { // 1
    return Scaffold(
      body: Center(
        child: Builder(
          builder: (context) => RaisedButton(
            child: Text("click me"),
            onPressed: () {
              Scaffold.of(context).showSnackBar(SnackBar( //2
                content: Text("hello"),
              ));
            },
          ),
        ),
      ),
    );
  }
}
```

在Scaffold中展示了一个SnackBar，我们使用Scaffold.of(context)获取ScaffoldState对象，失败了。 

咋一看，我们是在Scaffold节点的下面获取的，按理是可以的。

但，却发现context确实从AppState传进来的，而在在该层中context从节点树网上寻找，是找不到我们要的ScaffoldState的。

故，报错了。


flutter给了建议。

1. 使用 Builder，最简易的方法，将Button包裹一个Builder,该方法传入的context便能获取了。
2. 将RaisedButton写成一个独立的Widget，这么，我们在对应的Build方法中获取的context便能获取到Scaffold了。
3. 使用一个GlobalKey，将key绑定到Scaffold,那么之后调用 state.currentState便能获取到Scaffold了。


## 思考

每一个Widget都有自己的Context，对应Element树中位置。但是，问题是Widget没有直接获取Context的方法。为什么呢？因为仅在自己build方法
调用时，相应的context才确定，相比于在Widget中加入一个Context成员提供一个随时可能为空的Context，还不如在在build方法中提供一个
可靠的Context.

