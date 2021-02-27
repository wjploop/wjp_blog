---
title: Android: Single Activity design [(译)](https://proandroiddev.com/part-3-single-activity-architecture-514791724172)  
date: 2021-02-26 11:28:40  
tags:  翻译  
categories:
---

# Android: 单一Activity设计思路

在19年Google I/O Talk上，单一Activity设计原则伴随着 Jetpack Navigation 被提及，现在Google更推荐`单一Activity`设计作为首选架构。

> 我们又有造新名词了吗？"SAAs(Single Activity Applications)" 关于"单一Activity"，这个设计思路，即使Android开发圈子也并不是新鲜事物。

单一Activity设计，可以类比web开发中`单页面应用`,`单页面应用`设计思路在现代web开发框架中非常流行。

本文我们将讨论以下几点：

* `单一Activity`设计解决了什么问题？
* 何时该采用`单一Activity`设计?
* 对于现有的项目我们能做什么？

在讨论这几点之前，先简单介绍下Activity和Fragment  


## Activity是什么？

Activity作为Android应用的入口，显然在任何一个应用中至少需要一个Activity。

> Activity作为四大组件之一，聚焦于一点，描述用户能做什么。几乎所有的Activity都与用户交互，故`Activity`类负责创建`Widnow`展示
> 内容给用户，通过`setContentView(View)`方法我们可以放置所要展示的内容。尽管`Activity`通常用来展示全屏页面，但是也是可以用于
> 其他方式...
> 
> 摘自官方文档 https://developer.android.com/reference/android/app/Activity

## 使用Activity有什么问题？

嗯...并没有出现多大的问题，以致你一定要使用单一Activity设计。  

而且，Android设计的理念，一个页面对应了一个Activity。至少，在引入Fragment之前是这样的，这种理念从Android诞生之后就一直被
这么采用。

## Fragment是什么？

> Fragment能替代Activity，因为Activity能做的，Fragment也能做。

引入Fragment主要的目的，是让我们描述UI的代码可以复用，支持动态灵活的UI设计，因为Activity之间不能嵌套。（LocalActivityManager
已经永远地被遗弃了）。

Fragment的APIs推出后已经改进很多了。最初，Fragment只是拥有与Activity相似的生命周期，如今，Fragment已经支持自己`后退栈`，
并使用FragmentManager来管理。

在以下场景中，Fragment显得非常强大:

* 支持多屏适配
* 使用ViewPager，这要求一定要使用Fragment
* 创建一个包含UI库，暴露一个Fragment而不是一个Activity意义就大很多了

## Fragment存在什么问题?

Fragment曾存在很多问题，但随着Jetpack架构架构组件的引入，特别是`ViewModel`和`LiveData`的引入，很多问题都得到了解决。

实际上，抛开view-base框架，我们也毫无选择而必须使用Fragment。

> 在本文，我不会去讨论其他避免使用Fragment的方法，比如：[Conductor](https://github.com/bluelinelabs/Conductor)

Fragment有一个优势：它现在不是Android框架的一部分，而是存在于Jetpack中。这样，很容易利于Android团队修复问题和添加新特性。

因此，对于我们自己的app，若是想要展示UI，我们必须声明一个Activity作为入口，但对于实现其他页面，我们可以存在很多方案。

## 方案1：每一个页面作为独立的Activity

就如同Fragment未曾出现过一样。

比如，我们设计一个”购物”流程。我们可能会需要三个页面，购物清单清点页面，购物车详情页面，支付页面。这样就会有三个Activity需要被
创建：

1. OrderListReviewActivity
2. ShippingDetailActivity
3. PaymentActivity

## 方案2：每一个流程（模块）独立为一个Activity

App中每个模块独立为一个Activity，模块中的子页面使用Fragment来实现。

如此，将会有下列需实现：

1. CheckoutActivity
2. OrderListReviewFragment
3. ShippingDetailFragment
4. PaymentFragment

## 方案3：单一Activity

将只有一个Activity作为应用入口，所有的页面使用Fragment实现，这个Activity作为宿主负责存放和管理Fragment。

给个例子，当前流程跨平台框架都在使用该方案，比如 Xamarin, Ionic, Flutter, Reactive Native, 它们表现都很好。

通常，我看到的情况是，开始会选择方案1，只用Activity，慢慢地Fragment被引入，绝大数是因为：

1. 需要支持手机和平板
2. 需要复用UI
3. 需要使用ViewPager，这个强制使用Fragment
4. 使用了一些第三方库，其使用了Fragment作为暴露UI的方式，如:Google Maps

理论上，我们可能处于方案1和方案2之间，而方案2切换到单一Activity设计差别不大。

## 单一Activity设计解决了什么问题？

### 1. 不同版本Activity表现不一致

Activity作为Android框架一部分，其行为和支持特性绑定了Android版本。添加的新特性和修复的bug并不能保证在低版本系统中可用。
或许，你可能选择兼容库如 ActivityCompat,ActivityOptionCompat等来解决那些边角问题，但这很痛苦。

多个Activity不仅增加了开发时间，也增加了测试时间。表现不一致的问题，也同样存在于不同设备和不同版本。

###2. 在Activity功能共享数据

在Activity之间，想要共享数据，只能将该数据放在Application级别作用域中。然而，在该作用域中，其他的Android组件也能获取到了，
如Service，BroadcastReceiver,ContentProvider.

理想中，应该存在一个独立的作用域，用于存放几个页面共享的的数据，最好是在Activity级别的。

###3. 糟糕的用户体验

当切换Activity时，整个窗口都会被替换。因而，Toolbar/ActionBar也将会替换。我个人认为，Toolbar不应该被替换，而是应该更新相关
的内容即可。如同桌面应用一样，Toolbar永远不变，变的仅有下面的内容。

而且，由于不同版本的Activity表示不一致，其场景切换动画也会在不同设备不同版本表现不一致。

###4. 开发体验

当使用多个Activity时: 

* 每添加一个页面都要同步添加该Activity到Manifest文件中，仔细想想是不是有点奇怪。

* 某个控制功能需要在多个页面都需要实现时，将会耗费额外的精力。比如NavigationDrawer，底部栏，通用的菜单栏。

* 检测应用是否正在运行将变得困难，会发现在同一个时间，会有多个Activity在栈中。

单一Activity设计看来是一个很好的架构设计。但，直到现在，这样的架构缺少相关的框架，实现是困难的。目前存在一些支持单一Activity
的框架比如Conductor和Scoop,但它们不支持Fragment。

随着Navigation组件的引入，现在已经很容易实现单一Activity的架构了。

## 什么时候该采用单一Activity设计

采用单一Activity设计遇到的问题，Navigation组件已经为我们解决大部分了，所以，若是你现在开始新的项目，你应该毫无疑问的采用该设计。

Navigation组件提供了容易理解的API，类storyboard（IOS开发）的编辑器和详尽的文档和入门教程。

Navigation支持以下：

1. 处理[deep link](https://developer.android.com/training/app-links/deep-linking)
2. 简单可靠的转场动画
3. 更易用处理Toolbar
4. 支持动态特性模块

## 对于现有的项目我们能做什么？

理论上，对于那些已经混合了Activity和Fragment的现有项目，逐渐迭代到单一Activity设计是有意义的。开始，可以选择一个小模块进行
单一Activity多Fragment切换。  

若是有人拥有迭代现有项目到单一Activity设计的经验，我原意与之一起讨论其中的细节。

## 总结

* 单一Activity设计不是新鲜的概念了
* 诸如Conductor和Scoop基于View-based实现单一Activity的框架不支持Fragment
* 在Navigation推出之前，使用Fragment实现单一Activity设计不划算
* Navigation组件可实现基于Fragment的单一Activity
* 对于新项目，应该毫无疑问使用单一Activity设计了