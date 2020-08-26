---
title: Android View 是如何绘制的之二
tags: Android View Draw Graphics
published: true
layout: article
---

在[第一篇]({% post_url android/2020-07-15-how-android-view-draws-01 %})中，Android 如何从一开始执行到`View.onDraw(canvas)`方法已经介绍了，下边研究一下第二个问题： **调用 `View.onDraw(canvas)` 方法后，前后又执行了哪些动作，从而形成完整的绘制命令**

关于这个问题，从`View.updateDisplayListIfDirty()`方法开始，这个方法是调用`View.draw(canvas)`的开始。

<!--more-->

## View.draw(canvas)到底干了啥
要想知道`View.draw(canvas)`到底干了啥，还是从`View.updateDisplayListIfDirty()`方法入手。从代码中可以看出，在`view.draw(canvas)`之前：
1. 首先是通过`renderNode.start(width, height)`方法，获取了一个`DisplayListCanvas`
2. 然后调用在这个 `canvas` 上执行实际的绘制代码实现
3. 最后执行`renderNode.end(canvas)`
4. 和`setDisplayListProperties(renderNode)`方法。

```java
public RenderNode updateDisplayListIfDirty() {
    //...
    final DisplayListCanvas canvas = renderNode.start(width, height);
    draw(canvas);
    renderNode.end(canvas);
    setDisplayListProperties(renderNode);
}
```

### 1. renderNode.start(width, height)方法
首先`mRenderNode`实在 View 实例化的时候，就同时做了实例化。`renderNode.start(w,h)`方法实例化了一个和View等宽高的`DisplayListCanvas`，而且这个Canvas在实例化的时候，也会使用JNI方法，创建一个C++层的Canvas。源码如下：

```java
//RenderNode.java
public DisplayListCanvas start(int width, int height) {
    return DisplayListCanvas.obtain(this, width, height);
}
//DisplayListCanvas.java
private DisplayListCanvas(@NonNull RenderNode node, int width, int height) {
    super(nCreateDisplayListCanvas(node.mNativeRenderNode, width, height));
    mDensity = 0; // disable bitmap density scaling
}
```

<a name="create_canvas"></a>创建c++层的`Canvas`的过程：

```cpp
//canvas.cpp
Canvas* Canvas::create_recording_canvas(int width, int height, uirenderer::RenderNode* renderNode) {
    if (uirenderer::Properties::isSkiaEnabled()) {
        return new uirenderer::skiapipeline::SkiaRecordingCanvas(renderNode, width, height);
    }
    return new uirenderer::RecordingCanvas(width, height);
}
```

这个创建cpp层canvas的方法，会根据是否开启了`isSkiaEnabled()`，来返回`SkiaRecordingCanvas`和`RecordingCanvas`。

#### 是什么决定的skia是否开启呢
看代码就知道了：

```cpp
bool Properties::isSkiaEnabled() {
    auto renderType = getRenderPipelineType();
    return RenderPipelineType::SkiaGL == renderType || RenderPipelineType::SkiaVulkan == renderType;
}
RenderPipelineType Properties::getRenderPipelineType() {
    if (sRenderPipelineType != RenderPipelineType::NotInitialized) {
        return sRenderPipelineType;
    }
    char prop[PROPERTY_VALUE_MAX];
    property_get(PROPERTY_RENDERER, prop, "skiagl");
    if (!strcmp(prop, "skiagl")) {
        sRenderPipelineType = RenderPipelineType::SkiaGL;
    } else if (!strcmp(prop, "skiavk")) {
        sRenderPipelineType = RenderPipelineType::SkiaVulkan;
    } else {  //"opengl"
        sRenderPipelineType = RenderPipelineType::OpenGL;
    }
    return sRenderPipelineType;
}
```

可以从配置中取出`PROPERTY_RENDERER`的值，来决定使用何种渲染管道类型。默认是`skiagl`。 **那我就先从skiagl看起**。它的类继承关系是：

```cpp
class SkiaRecordingCanvas : public SkiaCanvas{}
class SkiaCanvas : public Canvas{}
```

### 2. draw(canvas)
所有的绘图操作就都是在`DisplayListCanvas`上，而从[上边的代码](#create_canvas)可以知道，底层创建的canvas是`SkiaRecordingCanvas`。

#### 2.1 绘制命令的开始
在继续向下看`SkiaRecordingCanvas`的绘制命令之前，先看看`View.draw(canvas)`中的一个方法，在绘制View的child的时候，会执行`ViewGroup.dispatchDraw(canvas)`方法，这个方法的源码：

```java
protected void dispatchDraw(Canvas canvas) {
    //canvas is DisplayListCanvas.
    boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);
    //...
    if (usingRenderNodeProperties) canvas.insertReorderBarrier();
}
//DisplayListCanvas.java
public boolean isRecordingFor(Object o) {
    return o == mNode;
}
public void insertReorderBarrier() {
    nInsertReorderBarrier(mNativeCanvasWrapper, true);
}
```

上边的代码中可以看到，当绘制的时候，一定会执行到`nInsertReorderBarrier()`方法，而这个方法，就会执行到`SkiaRecordingCanvas::insertReorderBarrier(true)`方法，开启一个`StartReorderBarrierDrawable`，[下边有介绍](#insertReorderBarrier)。

#### 2.2 记录绘制命令
`SkiaRecordingCanvas`做了些什么，我们就随便挑一个命令，就以`drawCircle(...)`为例，来看看吧。先看看关于`SkiaRecordingCanvas`的父类`SkiaCanvas`的介绍：
> A SkiaCanvas implementation that records drawing operations for deferred rendering backed by a SkLiteRecorder and a SkiaDisplayList.

```cpp
void SkiaRecordingCanvas::drawCircle(uirenderer::CanvasPropertyPrimitive* x,
                                     uirenderer::CanvasPropertyPrimitive* y,
                                     uirenderer::CanvasPropertyPrimitive* radius,
                                     uirenderer::CanvasPropertyPaint* paint) {
    drawDrawable(mDisplayList->allocateDrawable<AnimatedCircle>(x, y, radius, paint));
}

//SkiaCanvas.h
void drawDrawable(SkDrawable* drawable) { mCanvas->drawDrawable(drawable); }
```

这里的`mDisplayList`来自于创建canvas的时候，使用到的`RenderNode`。画圆命令的第一步就是：申请一个AnimateCircle 的Drawable，然后绘制这个Drawable。绘制是在SkCanvas上，也就是上边`drawDrawable(..)`方法中的`mCanvas`，这个Canvas是由[`skia`](https://github.com/google/skia)来提供的。

#### SkCanvas的实现是哪个
上边说到在绘制的时候，实际上是在`SkCanvas`上绘制，它是有`skia`这个2d图形库来提供的。那么在绘制的时候，提供的`SkCanvas`的实例是哪个呢？再看看`SkiaRecordingCanvas`的初始化工作：

```cpp
void SkiaRecordingCanvas::initDisplayList(uirenderer::RenderNode* renderNode, int width,
                                          int height) {
    if (renderNode) {
        mDisplayList = renderNode->detachAvailableList();
    }
    if (!mDisplayList) {
        mDisplayList.reset(new SkiaDisplayList());
    }
    mDisplayList->attachRecorder(&mRecorder, SkIRect::MakeWH(width, height));
    SkiaCanvas::reset(&mRecorder);
}
//SkiaCanvas.cpp
void SkiaCanvas::reset(SkCanvas* skiaCanvas) {
    if (mCanvas != skiaCanvas) {
        mCanvas = skiaCanvas;
        mCanvasOwned.reset();
        mCanvasWrapper.reset();
    }
    mSaveStack.reset(nullptr);
}
```

从上边的代码中可以看出，这个`SkCanvas`的实例，就是`mRecorder`，也就是`SkLiteRecorder.cpp`，来自skia库。具体的实现这里就略过了。至此就明白了，`view.draw(canvas)`方法，最终是把执行交给了`SkLiteRecorder`，在[开启Skia的时候](#是什么决定的skia是否开启呢)。

### 3 renderNode.end(canvas)
这个方法，具体代码如下：

```java
public void end(DisplayListCanvas canvas) {
    //调用finishRecording
    long displayList = canvas.finishRecording();
    //调用 native 的 nSetDisplayList(...)方法
    nSetDisplayList(mNativeRenderNode, displayList);
    //回收 canvas，并放到对象池中
    canvas.recycle();
}
```

共执行了三个过程，下面逐个介绍。

#### 3.1 canvas.finishRecording()
`SkiaRecordingCanvas::finishRecording`方法：<a name="insertReorderBarrier"></a>

```cpp
uirenderer::DisplayList* SkiaRecordingCanvas::finishRecording() {
    // close any existing chunks if necessary
    insertReorderBarrier(false);
    mRecorder.restoreToCount(1);
    return mDisplayList.release();
}
void SkiaRecordingCanvas::insertReorderBarrier(bool enableReorder) {
    if (nullptr != mCurrentBarrier) {
        // finish off the existing chunk
        SkDrawable* drawable =
                mDisplayList->allocateDrawable<EndReorderBarrierDrawable>(mCurrentBarrier);
        mCurrentBarrier = nullptr;
        drawDrawable(drawable);
    }
    if (enableReorder) {
        mCurrentBarrier = (StartReorderBarrierDrawable*)
                                  mDisplayList->allocateDrawable<StartReorderBarrierDrawable>(
                                          mDisplayList.get());
        drawDrawable(mCurrentBarrier);
    }
}
```

这个方法具体干什么的，不很清楚，只知道最后会绘制`EndReorderBarrierDrawable`，那就只能先看文档说明对这个类的介绍了。

> /**
> * See StartReorderBarrierDrawable.
> * EndReorderBarrierDrawable relies on StartReorderBarrierDrawable to host and sort the render
> * nodes by Z index. When EndReorderBarrierDrawable is drawn it will draw all render nodes in the
> * range with positive Z index. It is also responsible for drawing shadows for the nodes
> * corresponding to their z-index.
> */

从文档中可以看出，`SkiaRecordingCanvas::finishRecording`这个方法执行的时候，会把所有的`RenderNode`按照Z序列全部绘制，那么这个方法应该就是真正绘制命令执行的开始。但是源码中还有一个`StartReorderBarrierDrawable`是在一开始的时候开始绘制的，它跟`EndReorderBarrierDrawble`是一对，它是在[`View.draw(canvas)`的时候](#2-drawcanvas)调用开启的。根据以上文档和[SkiaRecordingCanvas](#22-记录绘制命令)的文档介绍，我们知道`SkiaRecodingCanvas`在`View.draw(canvas)`的时候，记录绘图操作，然后在`finishRecording`的时候，按照Z序列排序，并执行绘图操作。

#### 3.2 重置Canvas
`nSetDisplayList(...)`方法则是调用`SkiaRecordingCanvas::resetRecording()`方法，而这个方法，就是又重新初始化了DisplayList.

```cpp
//SkiaRecordingCanvas.h
virtual void resetRecording(int width, int height,
                                uirenderer::RenderNode* renderNode) override {
    initDisplayList(renderNode, width, height);
}
```

另外的`canvas.recycle()`更明显了，就是把这个`DisplayListCanvas`回收到对象池中。

## 总结
`View.draw(canvas)`前后到底干了啥。那就是：

1. 获取一个View对应的Canvas。
2. `View.draw(canvas)`->`ViewGroup.dispatchDraw(canvas)`->`canvas.insertReorderBarrier()`，底层会用到一个`StartReorderBarrierDrawable`。
3. `finishRecording`结束绘图命令的记录，底层会用到一个`EndReorderBarrierDrawble`，它和上一步中的Drawable配合使用（但具体做了什么，还未知），最终执行绘制命令，渲染图形。

这其中，对于绘图命令是如何记录和绘制的，下边再来更详细的研究。