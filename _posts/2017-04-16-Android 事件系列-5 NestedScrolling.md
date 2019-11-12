---
layout:     post
title:      Android 事件系列-5 NestedScrolling
date:       2017-04-16
author:     xflyme
header-img: img/post-bg-2017-04-16.jpg
catalog: true
tags:
    - Android
    - 事件分发
    - 源码分析
    - NestedScrolling
---


![图1](/img/event-5-1.gif)

先看这样一个效果图，这个效果图中有两级滑动。滑动开始时父 View 可滑动，滑动一定距离之后，父 View 不在继续上滑，此时子 View 接管滑动事件。

本系列的前两篇文章，我们学习了 Android View 和 Viewgroup 中的事件分发，从中我们知道如果一个控件决定接管一系列事件，那么这系列事件的 targetView 就会被确定，该系列事件的所有后续事件都会被分发给该 View 。

要实现上图中的效果也不是没有办法：
1. 父 View 先拦截事件。
2. 父 View 滑动到一定位置之后取消对该系列事件的拦截并生成一个 ACTION_DOWN 事件分发出去。

这种做法就是认为的把一系列事件拆分成两系列事件分发出去，父 View 消费一部分，子 View 消费另一部分。这种方法实践起来有一定困难，对开发者不是很友好。那是不是没有其他更好的办法了？Android 从 Lollipop 开始引进了一套全新的嵌套滑动机制 NestedScrolling，它可以完美的实现以上效果，今天我们来学习一下。

### NestedScrolling

NestedScrolling 机制主要包括以下四个对象：

```java
//接口
NestedScrollingParent
NestedScrollingChild

//帮助类
NestedScrollingChildHelper
NestedScrollingParentHelper                                                        
```
当然后续为了进一步对这一套机制进行完善又引入了 NestedScrollParent2 和 NestedScrollingChild2 接口。我们先来看一下源码，从源码重分析它的实现原理。NestedScrollView 既实现了 NestedScrollingChild2 也实现了 NestedScrollingParent 是我们很好的一个切入点，我们先从它看起。

NestedScrollView#onInterceptTouchEvent
```java
  @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        /*
         * This method JUST determines whether we want to intercept the motion.
         * If we return true, onMotionEvent will be called and we do the actual
         * scrolling there.
         */

       // 如果已经在拖拽，而且用户在移动他的手指 直接返回 true
        final int action = ev.getAction();
        if ((action == MotionEvent.ACTION_MOVE) && (mIsBeingDragged)) {
            return true;
        }

        switch (action & MotionEvent.ACTION_MASK) {
            case MotionEvent.ACTION_MOVE: {
                
                //进到这里说明 mIsBeingDragged 是 false

                //检查手指 id 是否合法
                final int activePointerId = mActivePointerId;
                if (activePointerId == INVALID_POINTER) {
                    // If we don't have a valid id, the touch down wasn't on content.
                    break;
                }
                //检查手指 index 是否合法
                final int pointerIndex = ev.findPointerIndex(activePointerId);
                if (pointerIndex == -1) {
                    Log.e(TAG, "Invalid pointerId=" + activePointerId
                            + " in onInterceptTouchEvent");
                    break;
                }

                final int y = (int) ev.getY(pointerIndex);
                final int yDiff = Math.abs(y - mLastMotionY);
                //判断是否已经达到垂直滑动的标准
                if (yDiff > mTouchSlop
                        && (getNestedScrollAxes() & ViewCompat.SCROLL_AXIS_VERTICAL) == 0) {
                    mIsBeingDragged = true;
                    mLastMotionY = y;
                    initVelocityTrackerIfNotExists();
                    mVelocityTracker.addMovement(ev);
                    mNestedYOffset = 0;
                    final ViewParent parent = getParent();
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                }
                break;
            }

            case MotionEvent.ACTION_DOWN: {
                final int y = (int) ev.getY();
                if (!inChild((int) ev.getX(), y)) {
                    mIsBeingDragged = false;
                    recycleVelocityTracker();
                    break;
                }

                /*
                 * Remember location of down touch.
                 * ACTION_DOWN always refers to pointer index 0.
                 */
                mLastMotionY = y;
                mActivePointerId = ev.getPointerId(0);

                initOrResetVelocityTracker();
                mVelocityTracker.addMovement(ev);
                /*
                 * If being flinged and user touches the screen, initiate drag;
                 * otherwise don't. mScroller.isFinished should be false when
                 * being flinged. We need to call computeScrollOffset() first so that
                 * isFinished() is correct.
                */
                // 如果正在 fling 同时用户触摸了屏幕 mIsBeingDragged 为 true，否则为 false
                mScroller.computeScrollOffset();
                mIsBeingDragged = !mScroller.isFinished();
                startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_TOUCH);
                break;
            }

            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                /* Release the drag */
                mIsBeingDragged = false;
                mActivePointerId = INVALID_POINTER;
                recycleVelocityTracker();
                if (mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0, getScrollRange())) {
                    ViewCompat.postInvalidateOnAnimation(this);
                }
                stopNestedScroll(ViewCompat.TYPE_TOUCH);
                break;
            case MotionEvent.ACTION_POINTER_UP:
                onSecondaryPointerUp(ev);
                break;
        }

        /*
        * The only time we want to intercept motion events is if we are in the
        * drag mode.
        */
        return mIsBeingDragged;
    }

```
在 ACTION_DOWN 的最后一行无论 mIsBeingDragged 是否为真，都调用了
startNestedScroll 我们进来看一下：

NestedScrollView#startNestedScroll
```java

    @Override
    public boolean startNestedScroll(int axes, int type) {
        return mChildHelper.startNestedScroll(axes, type);
    }
```
接着往下撸：
NestedScrollingChildHelper#startNestedScroll
```java


    /**
     * Start a new nested scroll for this view.
     *
     * <p>This is a delegate method. Call it from your {@link android.view.View View} subclass
     * method/{@link android.support.v4.view.NestedScrollingChild2} interface method with the same
     * signature to implement the standard policy.</p>
     *
     * @param axes Supported nested scroll axes.
     *             See {@link android.support.v4.view.NestedScrollingChild2#startNestedScroll(int,
     *             int)}.
     * @return true if a cooperating parent view was found and nested scrolling started successfully
     */
    public boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type) {
        if (hasNestedScrollingParent(type)) {
            // Already in progress
            return true;
        }
        if (isNestedScrollingEnabled()) {
            ViewParent p = mView.getParent();
            View child = mView;
            while (p != null) {
                if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {
                    setNestedScrollingParentForType(type, p);
                    ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);
                    return true;
                }
                if (p instanceof View) {
                    child = (View) p;
                }
                p = p.getParent();
            }
        }
        return false;
    }
```

关于该方法的返回值有这么一行注释：
> @return true if a cooperating parent view was found and nested scrolling started successfully

也就是说这个方法主要是一个找父亲的过程，目标是找到同样实现了嵌套滑动机制的父 View。该方法中有一个循环，直到找到实现了嵌套机制的父 View 才返回 true ，否则返回 false。还有一点需要注意：
> 该 ViewGroup 可以不是当前 View 的直接父 View。

如果找到了合适的父 View 该方法还做了两件事：调用父 View 的 onStartNestedScroll 和 onNestedScrollAccepted 方法。

ViewParentCompat#onStartNestedScroll
```java
    public static boolean onStartNestedScroll(ViewParent parent, View child, View target,
            int nestedScrollAxes, int type) {
        if (parent instanceof NestedScrollingParent2) {
            // First try the NestedScrollingParent2 API
            return ((NestedScrollingParent2) parent).onStartNestedScroll(child, target,
                    nestedScrollAxes, type);
        } else if (type == ViewCompat.TYPE_TOUCH) {
            // Else if the type is the default (touch), try the NestedScrollingParent API
            return IMPL.onStartNestedScroll(parent, child, target, nestedScrollAxes);
        }
        return false;
    }
```
ViewParentCompat#onNestedScrollAccepted
```java
  public static void onNestedScrollAccepted(ViewParent parent, View child, View target,
            int nestedScrollAxes, int type) {
        if (parent instanceof NestedScrollingParent2) {
            // First try the NestedScrollingParent2 API
            ((NestedScrollingParent2) parent).onNestedScrollAccepted(child, target,
                    nestedScrollAxes, type);
        } else if (type == ViewCompat.TYPE_TOUCH) {
            // Else if the type is the default (touch), try the NestedScrollingParent API
            IMPL.onNestedScrollAccepted(parent, child, target, nestedScrollAxes);
        }
    }

```

这两个方法比较简单，不在一一解释了，现在我们回到 onInterceptTouchEvent，上面我们看了 ACTION_DOWN，现在我们看下 ACTION_MOVE 主要代码是以下几行：

```java
  final int y = (int) ev.getY(pointerIndex);
                final int yDiff = Math.abs(y - mLastMotionY);
                if (yDiff > mTouchSlop
                        && (getNestedScrollAxes() & ViewCompat.SCROLL_AXIS_VERTICAL) == 0) {
                    mIsBeingDragged = true;
                    mLastMotionY = y;
                    initVelocityTrackerIfNotExists();
                    mVelocityTracker.addMovement(ev);
                    mNestedYOffset = 0;
                    final ViewParent parent = getParent();
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                }
```

先判断是否能达到触发滑动的标准，NestedScrollView 是垂直滑动，因此只判断了竖直上的滑动是否达标：
1. yDiff 是否大于 mTouchSlop 
2. 是否是垂直方向的滑动
如果这两项都为 true 的情况下则将 true 赋值给  mIsBeingDragged 同时禁止父类拦截滑动事件，而且还进行了一些初始化操作。

ACTION_CANCEL 和 ACTION_UP 主要是一些重置和停止嵌套滑动，就不一一解释了。

看完了事件拦截，我们看一下，嵌套滑动中是怎么处理滑动事件的：

NestedScrollView#onTouchEvent
```java
   @Override
    public boolean onTouchEvent(MotionEvent ev) {
        initVelocityTrackerIfNotExists();

        MotionEvent vtev = MotionEvent.obtain(ev);

        final int actionMasked = ev.getActionMasked();

        if (actionMasked == MotionEvent.ACTION_DOWN) {
            mNestedYOffset = 0;
        }
        vtev.offsetLocation(0, mNestedYOffset);

        switch (actionMasked) {
            case MotionEvent.ACTION_DOWN: {
                if (getChildCount() == 0) {
                    return false;
                }
                if ((mIsBeingDragged = !mScroller.isFinished())) {
                    final ViewParent parent = getParent();
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                }

                /*
                 * If being flinged and user touches, stop the fling. isFinished
                 * will be false if being flinged.
                 */
                if (!mScroller.isFinished()) {
                    mScroller.abortAnimation();
                }

                // Remember where the motion event started
                mLastMotionY = (int) ev.getY();
                mActivePointerId = ev.getPointerId(0);
                startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_TOUCH);
                break;
            }
            case MotionEvent.ACTION_MOVE:
                final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
                if (activePointerIndex == -1) {
                    Log.e(TAG, "Invalid pointerId=" + mActivePointerId + " in onTouchEvent");
                    break;
                }

                final int y = (int) ev.getY(activePointerIndex);
                int deltaY = mLastMotionY - y;
                if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset,
                        ViewCompat.TYPE_TOUCH)) {
                    deltaY -= mScrollConsumed[1];
                    vtev.offsetLocation(0, mScrollOffset[1]);
                    mNestedYOffset += mScrollOffset[1];
                }
                if (!mIsBeingDragged && Math.abs(deltaY) > mTouchSlop) {
                    final ViewParent parent = getParent();
                    if (parent != null) {
                        parent.requestDisallowInterceptTouchEvent(true);
                    }
                    mIsBeingDragged = true;
                    if (deltaY > 0) {
                        deltaY -= mTouchSlop;
                    } else {
                        deltaY += mTouchSlop;
                    }
                }
                if (mIsBeingDragged) {
                    // Scroll to follow the motion event
                    mLastMotionY = y - mScrollOffset[1];

                    final int oldY = getScrollY();
                    final int range = getScrollRange();
                    final int overscrollMode = getOverScrollMode();
                    boolean canOverscroll = overscrollMode == View.OVER_SCROLL_ALWAYS
                            || (overscrollMode == View.OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);

                    // Calling overScrollByCompat will call onOverScrolled, which
                    // calls onScrollChanged if applicable.
                    if (overScrollByCompat(0, deltaY, 0, getScrollY(), 0, range, 0,
                            0, true) && !hasNestedScrollingParent(ViewCompat.TYPE_TOUCH)) {
                        // Break our velocity if we hit a scroll barrier.
                        mVelocityTracker.clear();
                    }

                    final int scrolledDeltaY = getScrollY() - oldY;
                    final int unconsumedY = deltaY - scrolledDeltaY;
                    if (dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset,
                            ViewCompat.TYPE_TOUCH)) {
                        mLastMotionY -= mScrollOffset[1];
                        vtev.offsetLocation(0, mScrollOffset[1]);
                        mNestedYOffset += mScrollOffset[1];
                    } else if (canOverscroll) {
                        ensureGlows();
                        final int pulledToY = oldY + deltaY;
                        if (pulledToY < 0) {
                            EdgeEffectCompat.onPull(mEdgeGlowTop, (float) deltaY / getHeight(),
                                    ev.getX(activePointerIndex) / getWidth());
                            if (!mEdgeGlowBottom.isFinished()) {
                                mEdgeGlowBottom.onRelease();
                            }
                        } else if (pulledToY > range) {
                            EdgeEffectCompat.onPull(mEdgeGlowBottom, (float) deltaY / getHeight(),
                                    1.f - ev.getX(activePointerIndex)
                                            / getWidth());
                            if (!mEdgeGlowTop.isFinished()) {
                                mEdgeGlowTop.onRelease();
                            }
                        }
                        if (mEdgeGlowTop != null
                                && (!mEdgeGlowTop.isFinished() || !mEdgeGlowBottom.isFinished())) {
                            ViewCompat.postInvalidateOnAnimation(this);
                        }
                    }
                }
                break;
            case MotionEvent.ACTION_UP:
                final VelocityTracker velocityTracker = mVelocityTracker;
                velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
                int initialVelocity = (int) velocityTracker.getYVelocity(mActivePointerId);
                //如果速度大于 mMinimumVelocity 执行fling分发。
                if ((Math.abs(initialVelocity) > mMinimumVelocity)) {
                    flingWithNestedDispatch(-initialVelocity);
                } else if (mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0,
                        getScrollRange())) {
                    ViewCompat.postInvalidateOnAnimation(this);
                }
                mActivePointerId = INVALID_POINTER;
                endDrag();
                break;
            case MotionEvent.ACTION_CANCEL:
                if (mIsBeingDragged && getChildCount() > 0) {
                    if (mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0,
                            getScrollRange())) {
                        ViewCompat.postInvalidateOnAnimation(this);
                    }
                }
                mActivePointerId = INVALID_POINTER;
                endDrag();
                break;
            case MotionEvent.ACTION_POINTER_DOWN: {
                final int index = ev.getActionIndex();
                mLastMotionY = (int) ev.getY(index);
                mActivePointerId = ev.getPointerId(index);
                break;
            }
            case MotionEvent.ACTION_POINTER_UP:
                onSecondaryPointerUp(ev);
                mLastMotionY = (int) ev.getY(ev.findPointerIndex(mActivePointerId));
                break;
        }

        if (mVelocityTracker != null) {
            mVelocityTracker.addMovement(vtev);
        }
        vtev.recycle();
        return true;
    }
```

> 注意最后一行，这个方法直接返回 true

onTouchEvent 中最重要的是 ACTION_MOVE，我们重点分析一下。
重点是这一行，

```java
                if (this.dispatchNestedPreScroll(0, deltaY, this.mScrollConsumed, this.mScrollOffset, 0)) {

```
调用父 View 的 OnNestedPreScroll 让父 View 先去消费。如果父 View 没有消费完活着没有消费，把 mIsBeingDragged 置为 true 自己再消费。dispatchNestedPreScroll 之后的代码都是在处理自己怎么消费。我们知道这个逻辑就行了，不在钻牛角尖分析每一行代码。

在手指抬起的时候还有判断要不要执行 fling 操作，这个上面的代码中已经添加注释，也不再解释了。

上面我们主要分析的是 NestedScrollChild 以及 NestedScrollingChildHelper。

下面我们看一下 NestedScrollingParentHelper 和 NestedScrollParent。很多重要的操作，比如找到当前 View 的 NestedScrollParent 和事件的拦截都已经在 NestedScrollChild 和 NestedScrollingChildHelper 写好了。NestedScrollParent 中要做的事比较简单：
1. 是否要跟子 View 协同滑动。
2. 在协同滑动的时候自己要先消费多少距离。

代码比较简单，就不一一分析了。

### 总结

* NestedScrolling 能帮助开发者更简单（真的简单？在子 View 中处理事件拦截，逻辑一点都不少）的处理嵌套滑动。
* 子 View 中负责拦截事件，事件拦截之后先问问父 View 要不要先消费。
* 父 View 如果不消费，事件全部自己处理。
* 父 View 如果要消费，先将滑动的距离交给父 View 「享用」，父 View 剩下的，自己再处理。
