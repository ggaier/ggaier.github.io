---
title: ViewRootImpl 和 输入的关系
tags: Android Looper Advanced SourceDive View InputEvent
published: true
layout: article
---

[Android 消息机制的实现]({% post_url android/2020-06-14-android-looper-under-the-hood %})中提到输入事件是如何不阻塞主线程的。那就把这个问题先细分一下，再带着问题去学习：

1. 手指触摸在一个Activity上, 是由哪一部分开始接受输入事件的呢?
2. 输入事件产生后, 又是如何到了主线程的呢?
3. 这些输入事件, 触摸事件到底是如何产生的?
4. 输入事件产生后, 又是如何一步步的分发的呢?

先从第一个问题了解.

<!--more-->

## Activity中的输入事件最开始的入口是哪里

首先需要了解Activity中 View 的创建步骤. 才能了解入口在哪里. 代码的执行流程（基于android-9.0.0_r54）:

1. Activity启动的时候, 会首先通过`ApplicationThread.scheduleLaunchActivity`创建并调用Activity的attach, onCreate, onStart()生命周期
2. 调用Activity的`setContentView()`方法, 在onCreate的生命周期里. 这个方法会初始化DecorView.
3. 之后, AMS通过`scheduleShowWindow()`方法, 调用Activity.makeVisible()方法, 向Window中添加DecorView
4. `WindowManager.addView(mDecor...)`方法被调用, 通过`WindowManagerImpl.addView()`方法, 添加ContentView.
5. `WindowManagerGlobal.addView`方法中, 初始化ViewRootImpl, 并调用`ViewRootImpl.setView(mDecor)`方法, 初始化ViewRootImpl. 这个初始化方法中有初始化输入事件的方法.
6. `ViewRootImpl.setView()`方法会初始化`WindowInputEventReceiver`, 并传入`mInputChannel`和主线程的Looper
7. 输入事件通过`WindowInputEventReceiver.onInputEvent()`方法, 最终到达Activity的根View. 这也就是根View的入口
8. ViewRootImpl把接受到的事件, 加入队列, 并等待处理. 这是一个链表. `enqueueInputEvent()`
9. 取出链表中的表头, 也就是最旧的输入事件, 进行处理`deliverInputEvent()`
10. 不同的`InputStage`组成的责任链, 都得到一个处理InputEvent事件的方法.
11. 责任链中最后的一个是`ViewPostImeInputStage`, 它可以处理不同来源的输入事件, 比如触屏, 鼠标, TrackBall等等. 对于触屏设备来说, 输入事件的来源是`InputDevice.SOURCE_CLASS_POINTER`
12. `processPointerEvent`方法, 最终掉调用`mView.dispatchPointerEvent()`, 并处理它的返回值. 如果为`true`, 则表示已消费, 就不会给`InputStage`责任链的下一个对象处理了. 否则就继续转发给下一级InputStage.
13. `mDecorView.dispatchPointerEvent()`这个方法, 由于`DecorView`其实是一个`View`, 所以最终的入口就是`View.dispatchPointerEvent()`. 而这个方法最终通过`DecorView.dispatchTouchEvent()`处理.
14. `DecorView.dispatchTouchEvent()`方法, 最终通过`Window.Callback`回调到了`Activity.dispatchTouchEvent`

以上就是整个输入事件的入口流程了。

### 入口方法介绍

从上边的事件入口第 6 步了解到, 输入事件是由`ViewRootImpl`中的`WindowInputEventReceiver`接收到的. 所以要想解答这个问题, 就需要知道, 是谁调用了`WindowInputEventReceiver.onInputEvent()`方法。还是去`ViewRoomImpl`源码中找答案。

```java
public void public void setView(View view, ...){
    //...
    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
    //...
}
```

从上边的源码可以看到，在它的`setView(...)`方法中，会初始化一个`InputEventReceiver`，并传入`InputChannel` 和当前线程（主线程）的 looper。

#### InputEventReceiver.onInputEvent(event, displayId)

接着看`onInputEvent(...)`方法，这个方法底层的具体实现先不管，先看这个方法的文档注释：

```java
    /**
     * Called when an input event is received.
     * The recipient should process the input event and then call {@link #finishInputEvent}
     * to indicate whether the event was handled.  No new input events will be received
     * until {@link #finishInputEvent} is called.
     *
     * @param displayId The display id on which input event triggered.
     * @param event The input event that was received.
     */
    public void onInputEvent(InputEvent event, int displayId) {
        finishInputEvent(event, false);
    }
```

从方法注释中，可以得出，这就是当有输入事件的时候，最终回调到Activity 的入口。由于这之后，`InputEvent`事件都是在 ViewRootImpl 的主线程处理，所以可以确定，在接受输入事件的时候，主线程已经被唤醒了。所以就到一开始提到的第二个问题。

## 输入事件如何到了主线程

### 1. InputEventReceiver.java
继续从`InputEventReceiver`入手，看看它底下做了什么。从构造函数入手，源码是这样的：

```java
    /**
     * Creates an input event receiver bound to the specified input  channel.
     * @param inputChannel The input channel.
     * @param looper The looper to use when invoking callbacks.
     */
    public InputEventReceiver(InputChannel inputChannel, Looper looper) {
        //...
        mInputChannel = inputChannel;
        mMessageQueue = looper.getQueue();
        mReceiverPtr = nativeInit(new WeakReference<InputEventReceiver>(this), inputChannel, mMessageQueue);
    }
```

### 2. android_view_InputEventReceiver.cpp
那看来，核心就在`nativeInit(...)`方法了，看看android_view_InputEventReceiver.cpp里边的实现。

```c++
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject inputChannelObj, jobject messageQueueObj) {
            //...
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    sp<NativeInputEventReceiver> receiver = new NativeInputEventReceiver(env,
            receiverWeak, inputChannel, messageQueue);
    status_t status = receiver->initialize();
    //...
    receiver->incStrong(gInputEventReceiverClassInfo.clazz); // retain a reference for the object
    return reinterpret_cast<jlong>(receiver.get());
}
//初始化的方法
status_t NativeInputEventReceiver::initialize() {
    setFdEvents(ALOOPER_EVENT_INPUT);
    return OK;
}

void NativeInputEventReceiver::setFdEvents(int events) {
    if (mFdEvents != events) {
        mFdEvents = events;
        int fd = mInputConsumer.getChannel()->getFd();
        if (events) {
            mMessageQueue->getLooper()->addFd(fd, 0, events, this, NULL);
        } else {
            mMessageQueue->getLooper()->removeFd(fd);
        }
    }
}
```

看到这里就明白了，`setFdEvents(events)`方法，调用 MessageQueue 里的获取到 Looper，然后通过 Looper 里的 epoll添加 InputChannel 所对应的 fd。如此一来，就比较清晰了。

1. 以`ViewRootImpl.setView()`方法作为入口，它实例化了一个`WindowInputEventRecever`，并且构造方法带有两个参数，分别是`mInputChannel`，和`Looper.myLooper()`，前者暂时没有提到，后者是当前主线程的 looper。
2. `WindowInputEventReceiver`的构造方法，执行了`android_view_InputEventReceiver_nativeInit()`方法
3. 最终`NativeInputEventReceiver::setFdEvents()`方法，取到`inputChannel`背后的`fd`，并通过主线程的`Looper`添加到背后的`epoll`监听列表中。
4. 这样，当另外的进程或者线程向`inputChannel`中的 fd 写入数据后，主线程的Looper 背后的`epoll`由于监听了这个`fd`，会自动的从`epoll_wait`方法中中断，唤醒。这就是[Android 消息机制](./handler.md)提到的`epoll_wait`唤醒手段的第一种。

至此第二个问题的前半部分就比较清晰了，通过`epoll`来实现其他线程或者进程对当前应用主线程的唤醒。

但是也引出了一个新的问题，**`ViewRootImpl.setView()`方法中的这个`mInputChannel`又是谁向他写入输入事件的数据的呢？** 这个问题先暂且搁置，回头再说。现在继续往下说，设置了监听器，当输入事件唤醒了当前应用的主线程之后，后边又是怎么调用到了`WindowInputEventReceiver.onInputEvent(...)`方法的呢？我们继续寻找第二个问题的后半部分的答案。

### 3.把InputEvent传给InputEventReceiver.onInputEvent
再来接着第二步中的`setFdEvents()`方法，看看`Looper.addFd(int fd, int ident, int events, Looper_callbackFunc callback, void* data)`方法的签名。方法签名的第四个参数是一个callback，主线程被输入事件唤醒后，就会回调callback，被回调的方法是`handleEvent(int receiveFd, int events, void* data)`。

```c++
int NativeInputEventReceiver::handleEvent(int receiveFd, int events, void* data) {
    //...
    if (events & ALOOPER_EVENT_INPUT) {
        JNIEnv* env = AndroidRuntime::getJNIEnv();
        status_t status = consumeEvents(env, false /*consumeBatches*/, -1, NULL);
        mMessageQueue->raiseAndClearException(env, "handleReceiveCallback");
        return status == OK || status == NO_MEMORY ? 1 : 0;
    }
}
```

其他的方法的不关心，只关心一开始调用`nativeInit(...)`方法，其中设置的Event 类型`ALOOPER_EVENT_INPUT`，也就是输入事件。具体实现在`consumeEvents(...)`里。那就接着看该 cpp 文件下的`consumeEvents`方法。

```c++
status_t NativeInputEventReceiver::consumeEvents(JNIEnv* env,
        bool consumeBatches, nsecs_t frameTime, bool* outConsumedBatch) {
    //...
    bool skipCallbacks = false;
    for (;;) {
       //...
        status_t status = mInputConsumer.consume(&mInputEventFactory,
                consumeBatches, frameTime, &seq, &inputEvent, &displayId);
        if (status) {
            //...
        if (!skipCallbacks) {

            jobject inputEventObj;
            switch (inputEvent->getType()) {
                //...
            case AINPUT_EVENT_TYPE_MOTION: {
                if (kDebugDispatchCycle) {
                    ALOGD("channel '%s' ~ Received motion event.", getInputChannelName().c_str());
                }
                MotionEvent* motionEvent = static_cast<MotionEvent*>(inputEvent);
                if ((motionEvent->getAction() & AMOTION_EVENT_ACTION_MOVE) && outConsumedBatch) {
                    *outConsumedBatch = true;
                }
                inputEventObj = android_view_MotionEvent_obtainAsCopy(env, motionEvent);
                break;
            }

            default:
                assert(false); // InputConsumer should prevent this from ever happening
                inputEventObj = NULL;
            }
            if (inputEventObj) {
                env->CallVoidMethod(receiverObj.get(),
                        gInputEventReceiverClassInfo.dispatchInputEvent, seq, inputEventObj,
                        displayId);
                if (env->ExceptionCheck()) {
                    ALOGE("Exception dispatching input event.");
                    skipCallbacks = true;
                }
                env->DeleteLocalRef(inputEventObj);
            } else {
                //...
            }
        }

        if (skipCallbacks) {
            mInputConsumer.sendFinishedSignal(seq, false);
        }
    }
}
```

从代码中可以到，如果`mInputConsumer.consume(...&inputEvent)`方法成功取到`inputEvent`，就会在最后，调用 Java 层的方法`env->CallVoidMethod(receiverObj.get(), gInputEventReceiverClassInfo dispatchInputEvent, seq, inputEventObj,displayId)`，也就是`inputEventReceiver.dispatchInputEvent(int seq, InputEvent event, int displayId)`，这样，当主线程被输入事件唤醒后，最终就把事件从 cpp 层传到了Java层的`InputEventReceiver.dispatchTouchEvent(...)`方法了，这里最终会调用到`onInputEvent()`方法。

至此，一开始踢出的问题的前两个，就已经解决了。剩下的两个问题，后边再继续。来个流程小总结：

1. 主线程被 InputEvent 唤醒之后，会回调到`android_view_InputEventReceiver.cpp`中的`handlerEvent()`方法。
2. `handleEvent()`方法，会最终从`mInputConsumer.consume()`方法中，获取一个输入事件的数据。
3. 获取到的`InputEvent`数据，最终通过 JNI 调用 Java 层的方法，回调到`InputEventReceiver.dispatchTouchEvent(...)`方法。
4. 后续分发事件。涉及到了一开始提到的问题`4`

不过通过上边的步骤，自然就带来了下一问题， **`mInputConsumer.consume()`中的数据是怎么来的？** 这个问题，其实就是一开始的问题`3`了，后边再说。

学习了两个问题的同时，也带来了一个新的问题，这里记录下，后续再去接着解决这个问题：

* 5. **`ViewRootImpl.setView()`方法中的这个`mInputChannel`又是谁向他写入输入事件的数据的呢？**

其实这个问题适合第**3**个问题是关联的，那就是到底这些输入数据是如何写入到`mInputChannel`的，又如何被`mInputConsumer`取出来，然后分发到主线程的？下回说。



