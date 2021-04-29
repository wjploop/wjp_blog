---
title: Android-Transition
date: 2021-04-26 11:47:36
tags:
categories:
---

对Android Transition好奇，记录追踪代码

## Transition是什么？

<img src="/images/transitions.gif" width="240" height="200" />

如图，关注左上角的图块

可以看到，它先改变颜色，后平移到右下角，平移过程中大小也在变化，整体上看就是三个Animator的组合完成；
现在我们看到使用Transition如何实现的
```kotlin

        val transition = TransitionInflater.from(this).inflateTransition(R.transition.custom_transistion)
        root.setOnClickListener {
            // 每一次点击，往返切换start、end 状态
            TransitionManager.beginDelayedTransition(root, transition)
            end = !end
            toggleView()
        }

        // 区分当前状态，start or end
        var statusEnd = false

        fun toggleView() {
            container.children.forEach {
                it.visibility = if (statusEnd) View.INVISIBLE else View.VISIBLE
            }
            view_target.layoutParams = (view_target.layoutParams as FrameLayout.LayoutParams).apply {
                gravity = if (statusEnd) (Gravity.BOTTOM or Gravity.RIGHT) else (Gravity.TOP or Gravity.LEFT)
                width = if (statusEnd) 300 else 100
                height = if (statusEnd) 300 else 100
            }
            view_target.setBackgroundColor(if (statusEnd) Color.BLUE else Color.RED)
            view_target.apply {
                alpha = if(statusEnd) 1f else 0.5f
            }
        }
```
整体上看，依据 `statusEnd` 定义了两个状态 start，end，而依据这个状态，确定View的 visibility，position,
size,color的属性。  
切换状态，并非生硬地切换，而是在切换过程加入针对相关属性的过度动画，我想这就是Android加入这个库的用处吧。  

一个页面的两种状态，换个角度，也可以把这两种状态是视为两个页面，切换状态就是切换页面了，这里，我们将变化的页面称之为 `Scene`, 变化的过度动画交给
 `Transition` 处理。

### Scene

表示一个页面，会绑定到一个实际的View中。作为一个页面，会有进入和弹出两个动作，在这两个动作发生时会执行View
中某些property的动画。

### Transition

切换的涵盖了从一个旧的Scene到新的Scene的过程，包括 old scene exit 和 new scene enter.

维护一些动画信息，在Scene变化时执行持有的动画。
主要做两件事：
1. capture property value
2. play animations based on change to captured property value

## 还是跟着代码看看吧

先看看上面使用的使用到的 beginDelayedTransition()

>> 感觉注释写得很好了

方法名为意思为 启动一个延迟的切换， 为什么时延迟的呢？ 这个切换并没有马上执行，因为一个新的scene还没有确定呢。
而这个新的scene的创建过程也很有意思，它是依据下一帧与当前帧的“变化“确定的，而且它关注的只有参数中sceneRoot的ViewGroup节点下View的变化哦。
本来下一帧要确定好准备绘制的，但是遇到存在Transition未执行，会将其执行玩才真正绘制最终的下一帧。

换个说法，current frame >> next frame 之间，提取出 change in sceneRoot, 为这些 change 执行参数中的transition,即执行动画。
本质上，就是本来要绘制下一帧nextFrame的，发现有transition,则nextFrame要延后，在currentFrame与nextFrame之间插入一系列
的frame，而这些frame由transition来确定。

```java
    public static void beginDelayedTransition(@NonNull final ViewGroup sceneRoot, @Nullable Transition transition) {
        // 预备的执行的Transitions集合添加该某个View作为sceneRoot,
        // 限制了view只能能添加一次
        if (!sPendingTransitions.contains(sceneRoot) && ViewCompat.isLaidOut(sceneRoot)) {
            sPendingTransitions.add(sceneRoot);
            if (transition == null) {
                transition = sDefaultTransition;
            }
            final Transition transitionClone = transition.clone();
            // 停止当前运行的动画，记录当前帧的属性值、给scene机会执行一个可能需要exitAction
            sceneChangeSetup(sceneRoot, transitionClone);
            // 将当前sense设为空，即达到退出的效果
            Scene.setCurrentScene(sceneRoot, null);
            // 当scene变化时执行Transition，看看怎么监听sense变化的
            sceneChangeRunTransition(sceneRoot, transitionClone);
        }
    }
    private static void sceneChangeRunTransition(final ViewGroup sceneRoot, final Transition transition) {
        if (transition != null && sceneRoot != null) {
            MultiListener listener = new MultiListener(transition, sceneRoot);
            // 为View sceneRoot 添加attach、preDraw 状态的监听
            sceneRoot.addOnAttachStateChangeListener(listener);
            sceneRoot.getViewTreeObserver().addOnPreDrawListener(listener);
        }
    }

    
        // 当该View的脱离Window时先执行动画
        @Override
        public void onViewDetachedFromWindow(View v) {
            removeListeners();

            sPendingTransitions.remove(mSceneRoot);
            ArrayList<Transition> runningTransitions = getRunningTransitions().get(mSceneRoot);
            if (runningTransitions != null && runningTransitions.size() > 0) {
                for (Transition runningTransition : runningTransitions) {
                    runningTransition.resume(mSceneRoot);
                }
            }
            mTransition.clearValues(true);
        }

        // 在此刻，视图树上所有的View都已经测量，放置好位置，只是没有绘制
        // 在这个时间点，用户可以选择是否满意当前的布局，return true绘制当前帧， return false可以取消当前帧的绘制，

        @Override
        public boolean onPreDraw() {
            removeListeners();

        // 注意只是在第一次nextFrame操作，后续动画播放过程中的onPreDraw就不会处理了
        // Don't start the transition if it's no longer pending.
            if (!sPendingTransitions.remove(mSceneRoot)) {
                return true;
            }

            // Add to running list, handle end to remove it
            final ArrayMap<ViewGroup, ArrayList<Transition>> runningTransitions =
                    getRunningTransitions();
            ArrayList<Transition> currentTransitions = runningTransitions.get(mSceneRoot);
            ArrayList<Transition> previousRunningTransitions = null;
            if (currentTransitions == null) {
                currentTransitions = new ArrayList<>();
                runningTransitions.put(mSceneRoot, currentTransitions);
            } else if (currentTransitions.size() > 0) {
                previousRunningTransitions = new ArrayList<>(currentTransitions);
            }
            currentTransitions.add(mTransition);
            mTransition.addListener(new TransitionListenerAdapter() {
                @Override
                public void onTransitionEnd(@NonNull Transition transition) {
                    ArrayList<Transition> currentTransitions = runningTransitions.get(mSceneRoot);
                    currentTransitions.remove(transition);
                    transition.removeListener(this);
                }
            });
            mTransition.captureValues(mSceneRoot, false);
            if (previousRunningTransitions != null) {
                for (Transition runningTransition : previousRunningTransitions) {
                    runningTransition.resume(mSceneRoot);
                }
            }
            
            // 播放Transition,其实是 create animators and run 
            mTransition.playTransition(mSceneRoot);
                    // 适应 对比前后，创建很多animator
                >>  createAnimators(sceneRoot, mStartValues, mEndValues, mStartValuesList, mEndValuesList);
                >>  runAnimators();
            return true;
        }
    }
    
```

## 那启动一个Activity是如何执行Transition呢？


```
    // options 不为空，默认不传也会获取主题默认的，不过需要当前Window支持 Window.FEATURE_ACTIVITY_TRANSITIONS
    startActivityForResult(intent, -1, options);
    // 取消输入开始退出
    >> cancelInputsAndStartExitTransition
   
    
    public void startExitOutTransition(Activity activity, Bundle options) {
        mEnterTransitionCoordinator = null;
        if (!activity.getWindow().hasFeature(Window.FEATURE_ACTIVITY_TRANSITIONS) ||
                mExitTransitionCoordinators == null) {
            return;
        }
        ActivityOptions activityOptions = new ActivityOptions(options);
        if (activityOptions.getAnimationType() == ActivityOptions.ANIM_SCENE_TRANSITION) {
            int key = activityOptions.getExitCoordinatorKey();
            int index = mExitTransitionCoordinators.indexOfKey(key);
            if (index >= 0) {
                mCalledExitCoordinator = mExitTransitionCoordinators.valueAt(index).get();
                mExitTransitionCoordinators.removeAt(index);
                if (mCalledExitCoordinator != null) {
                    mExitingFrom = mCalledExitCoordinator.getAcceptedNames();
                    mExitingTo = mCalledExitCoordinator.getMappedNames();
                    mExitingToView = mCalledExitCoordinator.copyMappedViews();
                    mCalledExitCoordinator.startExit();
                }
            }
        }
    }
    
    public void startExit() {
        if (!mIsExitStarted) {
            backgroundAnimatorComplete();
            mIsExitStarted = true;
            pauseInput();
            ViewGroup decorView = getDecor();
            if (decorView != null) {
               // 禁止 layout乱入，layout也会导致 draw，而我们依赖着 draw前的时间点创建animator
                decorView.suppressLayout(true);
            }
            // 这个将共享元素移动到OverLay层，最高层哦
            moveSharedElementsToOverlay();
            startTransition(new Runnable() {
                @Override
                public void run() {
                    if (mActivity != null) {
                        beginTransitions();
                    } else {
                        startExitTransition();
                    }
                }
            });
        }
    }
    
    
    private void startExitTransition() {
        Transition transition = getExitTransition();
        ViewGroup decorView = getDecor();
        if (transition != null && decorView != null && mTransitioningViews != null) {
            setTransitioningViewsVisiblity(View.VISIBLE, false);
            TransitionManager.beginDelayedTransition(decorView, transition);
            setTransitioningViewsVisiblity(View.INVISIBLE, false);
            // 请求重绘，正好对应上了，onPreDraw
            // 否则没机会执行transition
            decorView.invalidate();
        } else {
            transitionStarted();
        }
    }
```
