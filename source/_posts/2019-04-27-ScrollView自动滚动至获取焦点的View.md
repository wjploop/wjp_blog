---
title: ScrollView自动滚动至获取焦点的View
date: 2019-04-27 15:14:27
tags: android
categories: 
- [code,android]
---

遇到了个好问题，以及一个好答案，记之

问：在xml布局中，ScrollView 里面包含一个RecyclerView，页面启动之后，直接滚动RecyclerView的位置了。
答：
分析流程：
1. 在Activity加载该xml布局后，通过`LayoutInflater.inflater(XmlPullParser parser, ViewGroup root, boolean attachToRoot)`.


加载并解析xml文件后，便开始构造View树。 rInflater(...)

其内部，是创建View，并递归加入到已有的View中。

addView（child，index，LayoutParams params）

其内部调用了addViewInner（child,index,params,preventRequestLayout) //addView之后，可以选择是否立即刷新当前布局的效果。

```
        //在添加一个拥有焦点的View时，递归为找到具体的view，其真有获有焦点
        final boolean childHasFocus = child.hasFocus();
        if (childHasFocus) {
            requestChildFocus(child, child.findFocus());
        }
```
 

看看requestChildFocus()
```java

//看到这个，可以直接阻止上层View获取焦点，这个蛮方便的
if (getDescendantFocusability() == FOCUS_BLOCK_DESCENDANTS) {
    return;
}

// We had a previous notion of who had focus. Clear it.
        if (mFocused != child) {
            if (mFocused != null) {
                mFocused.unFocus(focused);
            }

            mFocused = child;
        }      
        //请求
         if (mParent != null) {
            mParent.requestChildFocus(this, focused);
        }
        
@Override
public void requestChildFocus(View child, View focused) {
    if (DEBUG_INPUT_RESIZE) {
        Log.v(mTag, "Request child focus: focus now " + focused);
    }
    //会重绘
    checkThread();
    scheduleTraversals();
}

```
到此，可以知道，焦点会在加载之后确定。

另一个问题，为什么在Scrollview中，为什么会滚动到，获取的到焦点的View

在在ScrollView的重绘, onLayout()有，有一个mChildToScrollTo属性，
若是其不为空，会移动到该View位置。

而在requestChildFocus中，已经为其赋值。