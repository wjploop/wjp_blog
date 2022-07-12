---
title: ViewPager
date: 2019-08-11 09:53:02
tags: android

---

ViewPager 其作用是分页展示一个内容，现在Android主流应用布局，如微信，就是主页面是一个ViewPager，底部一个TabLayout构成。
用户可以左右滑动来翻页，页上下滚动的多用RecyclerView或ListView来实现，若是要实现左右翻动效果的，Android过去似乎并有一个正统的实现方式。
过去可以魔改ViewPager。不过，现在已经出了一个ViewPager2，它参考了RecyclerView的实现，翻动的可以是
传动的Fragment，还可以是View。这里，先看看传统ViewPager的实现部分。

> 什么时候使用适配器？为什么要使用？

我们需要展现一数据集，且这数据集的形式有多种，比如已经获取到的List,或是从数据库Cursor，若是不使用适配器，我们就需要分别展示数据的AListView，
ACursorView，这里的两个View都是可以展现对应数据集。而这里的问题在于，其实我们要展示的该类数据的样式，以及与该View的交互是一样了，即没有
能复用相同的逻辑。因而，为了不重复，引入适配器，ListView不再直接操作数据源，而是告知适配器我们需要什么操作，比如问问这个数据源总共有几个，
第i个的数据是什么，交互的，ListView还可以通知我们要或删除的一项了，而具体的添加，删除工作，就归Adapter来实现了。所以，**引入适配器是为了
复用展示和交互的逻辑。**


到此，我们确定三者，ViewPager负责展示数据以及交互，PageAdapter一个作为数据与View的中间桥梁，数据源。这里我们只要分析`PagerAdapter`，
`ViewPager`.

PagerAdapter是直接为与ViewPager交互的的接口，其实现有FragmentPagerAdapter,FragmentStatePagerAdapter.
两者有所不同应用场景，
FragmentPagerAdapter适合哪种页数固定，且页数少的，其创建后就要创建并保留所有的页面
FragmentStatePagerAdapter 适合页面较多的，因为该适配器不会去创建所有的页面，只会在需要时再去创建所需的，它还需负责保存和恢复过去的状态。


ViewPager中有个属性，mOffscreenPageLimit，默认为1，即ViewPager左右需要创建并保留多少个页面，比如，ViewPager会向Adapter预估要，
当前项 + 前后* mOffscreenPageLimit的数据。 


回到PagerAdapter,

继承它需要实现4个方法，可以说要回答是个问题。

instantiateItem(container,pos)  如何创建某项
destroyItem(container,pos)      如何删除某项
getCount()                      总共有几项
isViewFromObject(view,key)   确认page view是否已经关联上指定的key值

其简单版本， 
```kotlin
  view_pager.apply {

            adapter = object : PagerAdapter() {

                val sources = Array<String>(4) {
                    "page $it"
                }

                //创建一个View并添加人该其父视图，
                override fun instantiateItem(container: ViewGroup, position: Int): Any {
                    Log.d("wolf","instantiate pos:$position")
                    return TextView(container.context).apply {
                        text=sources[position]
                        container.addView(this)
                    }
                }
                //直接销毁既可
                override fun destroyItem(container: ViewGroup, position: Int, obj: Any) {
                    Log.d("wolf","destroy pos:$position")
                    if (container.getChildAt(position) == obj) {
                        container.removeView(obj as View?)
                    }
                }
                //这里直接就用该项的View作为key即ok，
                override fun isViewFromObject(view: View, obj: Any): Boolean {
                    val result=view==obj
                    Log.d("wolf", "isCreateFromObject: $result")
                    return result
                }

                override fun getCount(): Int {
                    return sources.size
                }

            }
        }
```

其实我们再看看FragmentPagerAdapter的实现，也是蛮简单的。
它的用的是FragmentManager来管理，
创建时，尝试通过 findFragmentByTag(tag)，
若找到，即已经添加fragmentManager了，直接attach（fragment),这会引发view的视图树变化
，若没有找到，即add(containerId,fragment,tag),
注意到是，这里的Fragment都是已经创建好点的，也就是说这里的instantiate和destroyItem仅仅attach和detach而已。
不过，注意到的是，这个还是设置了，setPrimaryItem,即设置当前要展示的页面，在设置这个页面之后，并调用了setUserVisibilityHint()方法，
通知Fragment此时的状态是否作显示



看看ViewPager，其直接继承了ViewGroup，那么我就看看三个方法吧，onMeasure(), onLayout(), onDraw()

> 先猜猜个大概，

其高度肯定是遍历子视图，找到最大的高度，
其宽度，在当前页处于首尾页时，加上一个mOffscreenLimit个的宽度，
否则，就加两个，
当然，不能缺了每个页面的margin。

放置时，所有子页面并排放即可。
绘制时，交个子页面绘制。

> 实际看看其实现

onMeasure()

因为我们会添加任意大小的View的，当然我们不希望其宽高随之变化，而是应该由父视图来决定其大小


...卒
