---
title: Android View 是如何绘制的-1
tags: Android View Draw Graphics
published: true
layout: article
---

首先以 Android 中的 View 为例，源码来自 Android9，它的绘制的入口是 `View.onDraw(canvas)`方法，但是要真正的了解 Android 绘制的流程，还需要解决几个问题：

1. <a name="question1"></a>View 是从哪里开始绘制的，并最终导向了`View.onDraw(canvas)`
2. 调用 `View.onDraw(canvas)` 方法后，后边又执行了哪些动作？
3. 底层实际的绘制管道`RenderingPipeline`是怎么实现的？
4. 最终 Android Graphics 的架构分层在整个流程中是如何体现的？

这篇文章就先针对第一个问题进行回答。

<!-- more -->

## 代码执行流程
下面代码的执行流程按照时间顺序来排列。

### 1. ViewRootImpl.invalidate()
调用 `View.invalidate()`方法， 最终会执行到`ViewRootImpl.invalidateRectOnScreen(dirty)`方法，然后通过`Choreographer.postCallback()`，安排在下一帧的时间执行一次绘制任务`perfromTraversals()`方法。

`ViewRootImpl.performTraversals()`这个方法，会依次执行`performMeasure()`，`perfromLayout()`和`performDraw()`方法。

### 2. ViewRootImpl.performDraw()

简单直接，从 ViewRoomImpl的代码`performDraw()`方法开始浏览, 它最终会调用到`draw(boolean)`方法，内部判断是否开启了硬件加速，如果开启了，就会启用`ThreadedRenderer`开启绘制，我们只浏览硬件加速开启后的绘制流程。具体的代码如下：

```java
if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
    if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
            //...
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this, callback);
    }
}
```

### 3. ThreadedRenderer.draw()
这个方法会首先更新 displaylist, 通过`updateRootDisplayList(view, callbacks)`方法；然后发起绘制，通过`nSyncAndDrawFrame(mNativeProxy, frameInfo, frameInfo.length)`方法。

第一个方法会执行到`ViewRootImpl.updateViewTreeDisplayList()`方法， 然后调用到了`View.updateDisplayListIfDirty()` 方法，其中这个 View 是`WindowManagerGlobal.addView()`中添加的`mDecorView`, 也就是 Activity 的rootView, 具体入口请参考`Activity.makeVisible()`方法。

关于`nSyncAndDrawFrame(...)`方法，本文不说，后续再说。

### 4. View.updateDisplayListIfDirty()
这个方法是更新 RootView视图树中的ChildView的 DisplayList, 顾名思义，就是如果`ChildView.isDirty()`, 才会去更新。最终会调用到dirtyView的`onDraw()`方法，同时`updateDisplayListIfDirty()`方法也会通过`RenderNode`记录`View.onDraw()`中的绘制命令，具体有什么用，会在后边**问题二**的文章中介绍。

```java
public RenderNode updateDisplayListIfDirty() {
    if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
            || !renderNode.isValid()
            || (mRecreateDisplayList)) {
        // Don't need to recreate the display list, just need to tell our
        // children to restore/recreate theirs
        if (renderNode.isValid()
                && !mRecreateDisplayList) {
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            //重新生成childView对应的DisplayList，会执行到view.draw(canvas)方法
            dispatchGetDisplayList();
            return renderNode; // no work needed
        }
        final DisplayListCanvas canvas = renderNode.start(width, height);
        try {
            //...
            draw(canvas);
        } finally {
            renderNode.end(canvas);
            setDisplayListProperties(renderNode);
        }
    }
}
```

至此我们知道了一个 View 的绘制开始的流程。

## 绘制流程总结
这篇文章的主要目的是为了解决文章一开始提到的[问题一](#question1)，比较简单，就是梳理了一下`View.draw(canvas)`方法被调用的上下文，方便了解一下View绘制的入口，并沿着绘制入口深入了解。绘制流程如下：

1. 通过 `View.invalidate()`方法，最终执行到`ViewRootImpl.invalidateRectOnScreen()`方法，然后ViewRootImpl通过Choreographer预定一个下一帧执行的绘制任务
2. Choreographer 通过监听底层的`VSync`事件，在显示器下一帧刷新的时候，执行绘制任务`performTraversals()`，`performDraw()`
3. `ViewRoomImpl.performDraw()`方法在判断需要硬件加速绘制的时候，通过`ThreadedRenderer.draw(...)`方法，更新`View`(RootView)对应的`Displaylist`，通过`view.updateDisplaylistIfDirty()`。
4. `View.updateDisplayListIfDirty()`方法，会最终执行到`view.onDraw(canvas)`方法。过程中有`RenderNode`参与。
