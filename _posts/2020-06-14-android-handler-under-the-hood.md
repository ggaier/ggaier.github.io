---
title: 消息机制到底是怎么实现的
tags: Android Handler Looper Advanced SourceCode
published: true
layout: article
mathjax: true
---

Android 消息机制中， 主要涉及Handler，MessageQueue，Looper，Thread。它们到底是怎么工作的呢？原理是什么呢？

<!--more-->

## 消息机制的工作流程

1. 创建一个线程.
2. 初始化一个Looper, Looper.myLooper().prepare(), 它会新建一个消息队列.
3. 循环Looper. `looper.loop()`进入死循环，并等待直到下一条消息时间到，然后从等待中恢复。
4. 外部调用者通过Handler(looper), `3`里的looper, 也可以发送消息到消息队列中, 并唤醒Looper线程.

那么问题来了，第`3`步中是如何在等待的时候不阻塞主线程的呢？又是如何响应手势事件的呢？

### 源码解析

#### Looper.java的初始化和循环

直接从上边的第二部开始, 通过`Looper.myLooper().prepare()`方法, 创建一个消息队列.

```java
```

`looper.loop()`方法通过, 无限循环, 从`looper.messageQueue()`中取出下一个消息. 由于这个是无限循环, 所以方法的核心点就是`messageQueue.next()`方法. 这个方法是一个阻塞式的方法, 它内部通过`epollwait()`方法, 等待一段时间, 等系统唤醒. 当系统唤醒之后, 就从消息队列里取出最近的一条消息, 然后对比时间, 如果时间到了, 就把消息发送给target, 让它们处理.

`MessageQueue.next()`的源码:

```java
for (;;) {
    if (nextPollTimeoutMillis != 0) {
        Binder.flushPendingCommands();
    }
    nativePollOnce(mPtr, nextPollTimeoutMillis);
    synchronized (this) {
        // Try to retrieve the next message.  Return if found.
        final long now = SystemClock.uptimeMillis();
        Message prevMsg = null;
        Message msg = mMessages;
        if (msg != null && msg.target == null) {
            // Stalled by a barrier.  Find the next asynchronous message in the queue.
            do {
                prevMsg = msg;
                msg = msg.next;
            } while (msg != null && !msg.isAsynchronous());
        }
        if (msg != null) {
            if (now < msg.when) {
                // Next message is not ready.  Set a timeout to wake up when it is ready.
                nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
            } else {
                // Got a message.
                mBlocked = false;
                if (prevMsg != null) {
                    prevMsg.next = msg.next;
                } else {
                    mMessages = msg.next;
                }
                msg.next = null;
                if (false) Log.v("MessageQueue", "Returning message: " + msg);
                msg.markInUse();
                return msg;
            }
        } else {
            // No more messages.
            nextPollTimeoutMillis = -1;
        }
}
```

#### 如何获取下一个消息

由于`MessageQueue`中的消息是按照时间先后顺序排列的一个链表, 所以取消息的原理就是, **等待两个消息的间隔时长**, 然后取出下一条消息. 这里就延伸出一个问题: **主线程是如何不阻塞的?**

先看下C++层如何实现的等待一段时间, 也就是`NativeMessageQueue.nativePollOnce()`方法的源码:

```c++
//执行wait操作, 等待一定的时间, 然后被系统唤醒.
//或者提前被mWakeEventFd唤醒. 当enqueueMessage()方法执行的时候.
 int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
 ...
  for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            ...
        }
  }
```

所以这个方法的核心是`epoll_wait`, 这个方法的描述是:

> epoll_wait -  wait  for  an I/O event on an epoll file descriptor

这个方法的最后一个参数, 指定了这个方法将会block的时长, 这个时长是由`CLOCK_MONOTONIC`来计算的. 这个方法将会一直阻塞, 直到:

* 一个文件描述符产生了一个事件
* 这个方法会被中断, 当有一个signal handler
* 或者超时.

##### 超时的计算

`epoll_wait`方法的官方文档中说超时的时间使用`CLOCK_MONOTONIC`计算的. Linux下有两种时钟, 另外一种是`CLOCK_REALTIME`. 前者表示的是相对任意的一个时间点的绝对的流逝时间, 不会随着系统时间变化, 它计算的不是当前的时间, 但是**确保时间的增加是严格的线性增加的, 所以用来计算时间只差非常的合适**.; 后者相对的是系统时钟的时间, 会跟随着系统时间的变化而变化.

##### Epoll的唤醒

上边说了, 除了超时会唤醒之外, 还有**文件描述符产生一个事件**的时候也会被唤醒. 所以看一下使用Android的looper中如何使用第一种方法唤醒. 下面是`Looper.cpp.wake()`方法的源码:

```c++
void Looper::wake() {
    uint64_t inc = 1;
    //这里向mWakeEventFd中写入一个一个8个字节的任意数据, 然后epoll_wait方法就会监听到事件变化,
    //进而不再继续阻塞.
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal: %s", strerror(errno));
        }
    }
}
```

另外, 还能中断`epoll_wait`方法的就是**signal**. signal是关于进程的信号, 它决定了这个进程的行为. 所以我们在应用中是不能使用该方法的.

### 主线程如何才没有被阻塞

看了上边的源码之后, 知道了主线程是通过阻塞一定的时间, 来获取下一条消息的, 所以如何才能在阻塞的时候, 随时的唤醒主线程呢?

了解了背后的epoll之后, 知道了主线程在取下一条消息的时候, 是阻塞式的. 阻塞的时间就是前后两条消息的时间间隔. 这个时候如何才能保证主线程有其他的事件的时候, 能够及时响应呢. 把这个问题分成两个问题:

1. 有新消息的时候, 怎么才能唤醒主线程?
2. 输入事件怎么办?

第一个问题, 由于新消息`Message`都是通过Handler想主线程的消息队列添加的, 而这个**添加消息的方法是在其他的线程执行的**. 所以尽管此时主线程阻塞, 仍然可以添加消息. 而添加消息后, 就会唤醒主线程. 请看上边的`Looper::wake()`源码.

第二个问题, 由于主线程的Looper背后依赖于epoll, 它用来监听多个`FileDescriptor`, 所以只要有一个产生了事件, 就会唤醒主线程. 而输入事件, 就是向主线程的Looper中, 添加了一个`FileDescriptor`. 所以当有输入事件产生的时候, 就会自动的唤醒主线程了. 具体的源码请参考: [input_01_viewrootimpl](./input_01_viewrootImpl.md)
