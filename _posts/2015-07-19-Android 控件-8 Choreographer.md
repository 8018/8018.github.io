---
layout:     post
title:      Android 控件-8 Choreographer
date:       2015-07-19
author:     xflyme
header-img: img/post-bg-2015-07-19.jpg
catalog: true
tags:
    - Android
    - 控件
    - 源码分析
---


### 简介
Android 从 4.1 开始引入 Choreographer，它可以看作是整个控件系统的「心脏」，它驱动着动画和 UI 绘制等，作用非常重要，我们今天来学习一下。

官网对其的介绍是：
> The choreographer receives timing pulses (such as vertical synchronization) from the display subsystem then schedules work to occur as part of rendering the next display frame.

翻译过来就是：choreographer 从显示子系统接收定时脉冲（比如垂直同步信号），然后在下一帧渲染时控制执行这些操作。

话不多说，我们直接来看源码。

### 构造函数
```java
 private Choreographer(Looper looper, int vsyncSource) {
        mLooper = looper;
        mHandler = new FrameHandler(looper);
        mDisplayEventReceiver = USE_VSYNC
                ? new FrameDisplayEventReceiver(looper, vsyncSource)
                : null;
        mLastFrameTimeNanos = Long.MIN_VALUE;

        mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());

        mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
        for (int i = 0; i <= CALLBACK_LAST; i++) {
            mCallbackQueues[i] = new CallbackQueue();
        }
    }

```
这里主要做一些初始化操作：
* 可以看到它需要一个 Looper，所以每个线程对应一个 Choreographer。
* 用对应的 Looper 创建了一个 Handler 用来做事件处理。
* 初始化 FrameDisplayEventReceiver ，FrameDisplayEventReceiver 用来接收垂直同步信号。
* 初始化 mLastFrameTimeNanos 用来记录上一帧事件，mFrameIntervalNanos 两帧之间的间隔时间，一般手机上为 1/60 s。
* 初始化 CallbackQueue 将会在下一帧渲染时找到合适的 callback 回掉。

### 成员

```java
  private final Object mLock = new Object();

    private final Looper mLooper;
    private final FrameHandler mHandler;

    // The display event receiver can only be accessed by the looper thread to which
    // it is attached.  We take care to ensure that we post message to the looper
    // if appropriate when interacting with the display event receiver.
    private final FrameDisplayEventReceiver mDisplayEventReceiver;
    //用来记录 callback 同时它也是一个链表，它持有下一个 callback 的引用。
    private CallbackRecord mCallbackPool;
    // callback 队列
    private final CallbackQueue[] mCallbackQueues;

    //
    private boolean mFrameScheduled;
    private boolean mCallbacksRunning;
    //上一帧时间
    private long mLastFrameTimeNanos;
    //每两帧之间的间隔
    private long mFrameIntervalNanos;
    private boolean mDebugPrintNextFrameTimeDelta;
```

其中有一个 FrameHandler 和 FrameDisplayEventReceiver 下面看一下它们俩的源码：

### FrameHandler

```java
private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                    doFrame(System.nanoTime(), 0);
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
```
比较简单，它只做三件事：
* MSG_DO_FRAME 处理下一帧的渲染
* MSG_DO_SCHEDULE_VSYNC 请求垂直同步信号
* MSG_DO_SCHEDULE_CALLBACK 执行 callback

> MSG_DO_SCHEDULE_VSYNC 之前这里一直认为是执行垂直同步，然后很多地方就说不通。如果是请求垂直同步信号，确实更能说的通，而且整个逻辑清晰了很多。

### FrameDisplayEventReceiver
```java
 private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;

        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            super(looper, vsyncSource);
        }

        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            //暂时不支持第二块屏幕，如果当前屏幕不是主屏，重新请求信号
            if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
                Log.d(TAG, "Received vsync from secondary display, but we don't support "
                        + "this case yet.  Choreographer needs a way to explicitly request "
                        + "vsync for a specific display to ensure it doesn't lose track "
                        + "of its scheduled vsync.");
                scheduleVsync();
                return;
            }

            // Post the vsync event to the Handler.
            // The idea is to prevent incoming vsync events from completely starving
            // the message queue.  If there are no messages in the queue with timestamps
            // earlier than the frame time, then the vsync event will be processed immediately.
            // Otherwise, messages that predate the vsync event will be handled first.
            long now = System.nanoTime();
            if (timestampNanos > now) {
                Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                        + " ms in the future!  Check that graphics HAL is generating vsync "
                        + "timestamps using the correct timebase.");
                timestampNanos = now;
            }

            if (mHavePendingVsync) {
                Log.w(TAG, "Already have a pending vsync event.  There should only be "
                        + "one at a time.");
            } else {
                mHavePendingVsync = true;
            }

            mTimestampNanos = timestampNanos;
            mFrame = frame;
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
    }
```
FrameDisplayEventReceiver 继承自 DisplayEventReceiver，主要用来接收垂直同步信号。当有垂直同步信号到来时，发送一个信息到 FrameHandler ，这个消息的回掉指向的是自己。当这个消息触发的时候，自己的 run() 方法将会被执行。

不过中间的一段英文注释我没看懂。

上面主要是接收垂直同步信号并处理，那么怎么请求垂直同步信号？

Choreographer#postCallback
```java
public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }
```

Choreographer#postCallbackDelayed
```java
public void postCallbackDelayed(int callbackType,
            Runnable action, Object token, long delayMillis) {
        if (action == null) {
            throw new IllegalArgumentException("action must not be null");
        }
        if (callbackType < 0 || callbackType > CALLBACK_LAST) {
            throw new IllegalArgumentException("callbackType is invalid");
        }

        postCallbackDelayedInternal(callbackType, action, token, delayMillis);
    }
```
Choreographer#postCallbackDelayedInternal
```java
 private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        if (DEBUG_FRAMES) {
            Log.d(TAG, "PostCallback: type=" + callbackType
                    + ", action=" + action + ", token=" + token
                    + ", delayMillis=" + delayMillis);
        }

        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            //添加一个指定类型的 callback 到 CallbackQueue
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            // 如果请求的时间小于等于当前时间，立即执行 scheduleFrameLocked 
            // 否则发送一个延时到 handler
            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
```
注意上面做了一个时间判断，不过不是立即执行的只是通过 Handler 做了一个延时处理，最后都是走的 scheduleFrameLocked。

Choreographer#scheduleFrameLocked
```java

    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            if (USE_VSYNC) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }

                // If running on the Looper thread, then schedule the vsync immediately,
                // otherwise post a message to schedule the vsync from the UI thread
                // as soon as possible.
                // 如果当前线程是主线程，立即请求垂直同步事件
                if (isRunningOnLooperThreadLocked()) {
                    scheduleVsyncLocked();
                } else {
                //当前线程不是主线程，发送消息到 Handler，注意这个消息是添加到队列最前面
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                // 不使用垂直同步的情况下，直接通过 handler 机制执行下一帧
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
                }
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
```
Choreographer#scheduleVsyncLocked
```java

    private void scheduleVsyncLocked() {
        mDisplayEventReceiver.scheduleVsync();
    }
```
DisplayEventReceiver#scheduleVsync
```java
    /**
     * Schedules a single vertical sync pulse to be delivered when the next
     * display frame begins.
     */
    public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
            nativeScheduleVsync(mReceiverPtr);
        }
    }

```
这里最终调用了 native 方法，不过看其注释应该是请求垂直同步事件。
请求垂直同步事件的流程已经走完，当垂直同步事件来的时候是怎么执行的？
下面看一下执行流程。

Choreographer#doFrame
```java
void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
        // 是否有 callback 需要执行
        // postCallBack 的时候置为 true
        // 执行 frame 时置为 false
            if (!mFrameScheduled) {
                return; // no work to do
            }

            // 打印跳帧时间
            if (DEBUG_JANK && mDebugPrintNextFrameTimeDelta) {
                mDebugPrintNextFrameTimeDelta = false;
                Log.d(TAG, "Frame time delta: "
                        + ((frameTimeNanos - mLastFrameTimeNanos) * 0.000001f) + " ms");
            }
            // 垂直同步信号时间
            long intendedFrameTimeNanos = frameTimeNanos;
            //当前时间
            startNanos = System.nanoTime();
            //时间差
            final long jitterNanos = startNanos - frameTimeNanos;
            //时间差大于两帧间隔说明可能跳帧
            if (jitterNanos >= mFrameIntervalNanos) {
                //跳过的帧数
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                //跳帧超过一定数目，打印警告信息
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                    Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                            + "The application may be doing too much work on its main thread.");
                }
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
                if (DEBUG_JANK) {
                    Log.d(TAG, "Missed vsync by " + (jitterNanos * 0.000001f) + " ms "
                            + "which is more than the frame interval of "
                            + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                            + "Skipping " + skippedFrames + " frames and setting frame "
                            + "time to " + (lastFrameOffset * 0.000001f) + " ms in the past.");
                }
                //修正偏差
                frameTimeNanos = startNanos - lastFrameOffset;
            }
            //当前帧的时间小于上一帧的时间，这一帧将不会被执行，并且要重新请求垂直同步信号
            if (frameTimeNanos < mLastFrameTimeNanos) {
                if (DEBUG_JANK) {
                    Log.d(TAG, "Frame time appears to be going backwards.  May be due to a "
                            + "previously skipped frame.  Waiting for next vsync.");
                }
                scheduleVsyncLocked();
                return;
            }
             //记录当前frame信息        
            mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
            mFrameScheduled = false;
            //记录上一次frame开始时间
            mLastFrameTimeNanos = frameTimeNanos;
        }

        try {
            //执行相关 callback
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

            mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }

        if (DEBUG_FRAMES) {
            final long endNanos = System.nanoTime();
            Log.d(TAG, "Frame " + frame + ": Finished, took "
                    + (endNanos - startNanos) * 0.000001f + " ms, latency "
                    + (startNanos - frameTimeNanos) * 0.000001f + " ms.");
        }
    }
```
doCallbacks 方法暂时不看了，至此请求垂直同步信号，接收信号并处理同步都已经看完。下面是流程图：

![图一](/img/android-view-8-1.png)
