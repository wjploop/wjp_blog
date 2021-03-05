---
title: Fragment Transactions & Activity State Loss (译)  
date: 2021-03-05 11:13:31  
tags: 翻译  
categories:  
---

> [原文](https://www.androiddesignpatterns.com/2013/08/fragment-transaction-commit-state-loss.html)


# Fragment Transactions 和 Activity 状态丢失


自Android3.0后，以下报错信息在StackOverflow困扰众人许久了

>java.lang.IllegalStateException: Can not perform this action after onSaveInstanceState
at android.support.v4.app.FragmentManagerImpl.checkStateLoss(FragmentManager.java:1341)
at android.support.v4.app.FragmentManagerImpl.enqueueAction(FragmentManager.java:1352)
at android.support.v4.app.BackStackRecord.commitInternal(BackStackRecord.java:595)
at android.support.v4.app.BackStackRecord.commit(BackStackRecord.java:574)

本文将要解释该异常为什么会出现，以及何时会出现，随便给出几个意见来处理该问题。


## 该异常为什么会发生？

该异常抛出的原因是，在Activity状态已经保存后仍试图提交一个FragmentTransaction，其导致了一个现象被称之为状态丢失。在探究其中的
各种细节之前，我们先来看看`onSaveInstanceState`方法调用时发生了什么。正如在我的上一篇文章
-[Binders & Death Recipients](https://www.androiddesignpatterns.com/2013/08/binders-death-recipients.html) -所说,
在Android的运行环境中，App对自己的命运掌控甚少。Android系统在内存不足的情况下可以在杀死进程，导致后台Activity在未收到任何通知下
被杀掉。为了避免用户察觉的到这种不稳定的问题（切换到后台被杀或不被杀，当切换到前台时表现应该一致），框架给了Activity一个机会，让它在
可能被销毁的情况下，使用`onSaveInstanceState()`方法来保存好当前的状态。当App由后台切换到前台，恢复之前保存的状态，无论App是否
被杀与否，用户感知到的都是一样的。

当系统回调`onSaveInstanceState()`方法时，它将一个`Bundle`对象传给Activity用于保存Dialog,Fragment,Views的状态，在该方法
返回后，系统序列化该Bundle数据（通过parcel序列化）通过Binder传给系统服务进程（System Server process）,使其能安全保存着。之后
当系统重建创建Activity时，又将该Bundle传回给应用来恢复原来的状态。

那为什么该异常会抛出呢？该问题源自于该Bundle数据代表Activity在调用`onSaveInstance()`这一时刻的快照，这意味着在
`onSaveInstance()`之后调用`FragemntTransaction#commit()`，这个transaction将不会被保存。而从用户的角度上看，会导致UI的
数据混乱。为了保证用户体验，Android只好抛异常了避免状态丢失了。


## 何时该异常会抛出？

假如你之前遇到该异常，你可能会注意到该异常抛出的时间点在不同版本上会有所不同。比如，你可能注意到在老设备上更容易抛出该异常，或是
你的应用可能更容易崩溃在你使用support库而非官方库时。这些细小的不一致行为导致很多人认为support是存在bug不能被信任，而这些观点，
通常是错误的。

这些不一致行为，原因是在在Android3.0之后，Activity的生命周期方法发生了一些重大改变。3.0之前，Activity是在paused之后就可以
被系统认为可杀掉的，意味着在`onPause()`方法之前会保证调用`onSaveInstance()`。而在3.0之后，Activity是要在stopped状态后
才会是可杀的，也就是会在`stoped`之前保证保存状态。

由于Activity生命周期的变化，支持库需要兼容不同的平台版本。比如，该异常会在`onSaveInstanceState()`方法后执行`commit()`便会
抛出，以此提示开发者状态丢失了。而`onSaveInstanceState()`调用时机在3.0之前更早一些，也就导致低版本更容易状态丢失。为了能够在
支持不同版本，Android团队做了妥协：允许在低版本上Android中，在`onPause()`和`onStop()`之间提交commit会导致异常。支持库在
不同版本表现如下表格：


| | 3.0前 | 3.0后 |
| --- | :---: | :---: |
 在 onPause()前 commit() | OK | OK |
 在 onPause() 和 onStop() 之间commit() | 状态丢失 | OK |
在 onStop() 之后commit() | Exception | Exception |

## 如何避免该异常

一旦理解状态丢失是如何发生之后，避免该异常就容易多了。若是你在阅读本文之前就理解了，也希望你可以通过本文知道支持库是如何工作的以及
为什么App中避免状态丢失如此重要。若是你是为了寻找一个快速的解决方案而看到本文，这里有几个建议希望能够对你处理FragmentTransaction
有帮助。


### 在Activity生命周期提交事务（commit transaction）时需要谨慎

大部分应用只会在onCreate()方法中或是响应用户操作时才会提交事务，而这不会出现问题。然而，在其他的生命周期方法开始事务时，事情就会
变得复杂了，比如`onActivityResult()`,`onStart()`,`onResume()`. 比如，你不应该在`onResume()`方法中提交事务，因为有可能
此时Activity的状态没有正确回复（restored）,详见
[文档](https://developer.android.com/reference/android/support/v4/app/FragmentActivity.html#onResume()),
> 文档内容（译者加）：分发 onResume() 方法给Fragment。注意，为了更好的兼容低版本平台，该activity中attached的Fragment并没有
> 进入resumed状态。这意味着原来保存的状态（若是以前保存有）没有恢复（原来的状态还是在bundle中，而非在当前state中），当前是不
> 允许提交事务修改状态的，你应该在`onResumeFragments()`方法中提交事务修改。
> 
如果你需要在onCreate()之外的生命周期中提交事务，应该在`FragmentActivity#onResumeFragemnts()`或`Activity#onPostResume()`。
这两个方法保证原来的状态已经正确的恢复，因此可以避免状态丢失的可能性。（若是想要在`Activity#onActivityResult()`方法中提交
事务，可以参看我的StackOverFlow中的
[回答](https://stackoverflow.com/questions/16265733/failure-delivering-result-onactivityforresult) ）

### 避免在异步的回调方法中提交事务

这比如常用的方法`AsyncTask#onPoastExecute()`和`LoadManager.LoaderCallback#onLoadFinished()`. 问题在于，当在这些方法
提交事务时，我们并不知道当前Activity所处的状态。如下展示下异常出现的过程。

1. Activity中开始执行一个`AsyncTask`
2. 用户点击Home键，导致Activity的`onSaveStateInstance()`和`onStop()`执行
3. `AsyncTask`任务完成执行`onPostExecutes()`,并不意识到Activity已经 stopped
4. 在`onPostExecutes()`中提交事务，导致异常抛出

通常来说，最好不在异步回调中提交事务。谷歌工程师看起来也认同这一原则，在Android开发团队的一篇
[文章](https://groups.google.com/d/msg/android-developers/dXZZjhRjkMk/QybqCW5ukDwJ) 中，他们认为在异步回调中
执行提交事务会导致突然切换界面，导致糟糕的用户体验。若是你的App一定要在异步回调中提交事务，并没有容易的方法来保证提交事务在
保存状态前执行，你可能需要使用`commitAllowStateLoss()`并自己出现状态能会丢失的情况。(StackOverFlow有两篇文章可以参考，
[1](https://stackoverflow.com/questions/8040280/how-to-handle-handler-messages-when-activity-fragment-is-paused)
[2](https://stackoverflow.com/questions/7992496/how-to-handle-asynctask-onpostexecute-when-paused-to-avoid-illegalstateexception)

### 使用commitAllowStateLoss 应当作为最后选项

`commit()`和`commitAllowStateLoss()`两者唯一区别在于后者在发现状态状态可能丢失时，不会抛异常。通常你也不想用该方法，因为这
意味这要承受状态丢失的可能。最好的解决方案当然是保证commit在保存状态前执行，这也保证好的用户体验。除非状态丢失无法避免，否则
`commitAllowStateLoss`不改使用。

希望这些建议能够解决你遇到的问题。若是你仍遇到问题，在StackOverFlow提交问题并在下面评论区留下链接，我会帮忙看看滴。

总之，感谢你的阅读。若是有问题欢迎评论，别忘了点赞分享~

> 上次更新于2014-1-8
> 译者翻译与2021-3-5，哭了~，陈年好文啊