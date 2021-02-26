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
* 创建