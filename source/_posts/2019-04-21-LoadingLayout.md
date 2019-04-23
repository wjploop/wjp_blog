---
title: LoadingLayout
date: 2019-04-21 07:27:38
tags: 
- android
- 自定义View

categories: 
- [code, android]

---

### 自定义LoadingLayout的笔记

在学习[SmartRefreshLayout](https://github.com/scwang90/SmartRefreshLayout)库当中看到了一个
[LoadingLayout](https://github.com/czy1121/loadinglayout), 用于在请求数据时，状态改变，
如 `empty` > `loading` > `error` > `normal`。日常的app还是会经常使用的，每次实现很麻烦，所以将其维护成一个库还是挺好的（开发效率为王）。
看了作者的实现之后，发现其中确实有些东西是很值得学习，原版实现较简单，因而维护先自己的版本。（方便自己的整理思路）




### 类库描述

作为一个View，内部维护有4种状态：
1. 空白   
2. 加载中 
3. 错误
4. 正常内容

对外提供修改状态，页面展现相应的内容页面，创建`LoadingLayout`中，提供对状态对应页面的展现配置。
提供了在style.xml配置，或在代码中配置。配置`normal`状态没有意义，所以可以为其余三个状态配置对应的资源文件。
`normal`状态的是处于`LoadingLayout`的子视图，配置这个关系，可以在xml文件中配置，也提供了`wrap(view)`,
'wrap(activity)'方法手动配置，无需修改xml文件。 
 
由于空白页，错误页都是常见的，一般会由一个图片和描述组成，项目不同只要修改相应的图片和文字，所以，
类库提供了直接配置相应图片，文字。在错误页提供了"重新加载接口"。

### 具体实现

利用FrameLayout来存放各个状态的View，当需要展现特定View时，set all view gong and set target 
view to visible.

更具体的说，内部维护了Map<resId,View>，并且该数据时懒加载的，在第一次加载对应View时inflate相应的ResId，
并对加载的View配置相应的属性（如图片，文字）

遇到了一个问题：
在配置style文件中的dimension中，注意引用指向引用的问题，有种指针的味道。