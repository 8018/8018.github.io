---
layout:     post
title:      Android 事件系列-4 ViewDragHelper
date:       2017-04-09
author:     xflyme
header-img: img/post-bg-2017-04-09.jpg
catalog: true
tags:
    - Android
    - 事件分发
    - 源码分析
    - ViewDragHelper
---


### 前言
自定义 ViewGroup 有时候需要对子 View 进行拖拽处理，Android 提供了一个很好的工具类 ViewDragHelper，本节我们学习一下它。

### Demo
我们先看一个 Demo，首先定义一个 ViewGroup，它继承了 LinearLayout 源码如下：
```java
public class DragLayout extends LinearLayout {

    static final String TAG = "DragLayout";

    private ViewDragHelper mDragger;

    private ViewDragHelper.Callback callback;

    private ImageView iv1;
    private ImageView iv2;

    private Point initPointPosition = new Point();

    @Override
    protected void onFinishInflate() {
        iv1 = (ImageView) this.findViewById(R.id.iv1);
        iv2 = (ImageView) this.findViewById(R.id.iv2);
        super.onFinishInflate();

    }

    public DragLayout(Context context) {
        super(context);

    }

    public DragLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
        callback = new DraggerCallBack();
        //第二个参数就是滑动灵敏度的意思 可以随意设置
        mDragger = ViewDragHelper.create(this, 1.0f, callback);
    }

    class DraggerCallBack extends ViewDragHelper.Callback {

        //这个地方实际上函数返回值为true就代表可以滑动 为false 则不能滑动
        @Override
        public boolean tryCaptureView(View child, int pointerId) {
            if (child == iv2) {
                return false;
            }
            return true;
        }


        //这个地方实际上left就代表 你将要移动到的位置的坐标。返回值就是最终确定的移动的位置。
        // 我们要让view滑动的范围在我们的layout之内
        //实际上就是判断如果这个坐标在layout之内 那我们就返回这个坐标值。
        //如果这个坐标在layout的边界处 那我们就只能返回边界的坐标给他。不能让他超出这个范围
        //除此之外就是如果你的layout设置了padding的话，也可以让子view的活动范围在padding之内的.

        @Override
        public int clampViewPositionHorizontal(View child, int left, int dx) {
            //取得左边界的坐标
            final int leftBound = getPaddingLeft();
            //取得右边界的坐标
            final int rightBound = getWidth() - child.getWidth() - getPaddingRight();
            //这个地方的含义就是 如果left的值 在leftbound和rightBound之间 那么就返回left
            //如果left的值 比 leftbound还要小 那么就说明 超过了左边界 那我们只能返回给他左边界的值
            //如果left的值 比rightbound还要大 那么就说明 超过了右边界，那我们只能返回给他右边界的值
            return Math.min(Math.max(left, leftBound), rightBound);
        }

        //纵向的注释就不写了 自己体会
        @Override
        public int clampViewPositionVertical(View child, int top, int dy) {
            final int topBound = getPaddingTop();
            final int bottomBound = getHeight() - child.getHeight() - getPaddingBottom();
            return Math.min(Math.max(top, topBound), bottomBound);
        }

        @Override
        public void onViewReleased(View releasedChild, float xvel, float yvel) {
            //松手的时候 判断如果是这个view 就让他回到起始位置
            /*if (releasedChild == iv1) {
                //这边代码你跟进去去看会发现最终调用的是startScroll这个方法 所以我们就明白还要在computeScroll方法里刷新
                mDragger.settleCapturedViewAt(initPointPosition.x, initPointPosition.y);
                invalidate();
            }*/
        }
    }

    @Override
    public void computeScroll() {
        if (mDragger.continueSettling(true)) {
            invalidate();
        }
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        return super.dispatchTouchEvent(ev);
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);
        //布局完成的时候就记录一下位置
        Log.e(TAG,"onLayout");
        initPointPosition.x = iv1.getLeft();
        initPointPosition.y = iv1.getTop();
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {

        switch (ev.getAction()){
            case MotionEvent.ACTION_DOWN:

                break;
        }

        //决定是否拦截当前事件
        return mDragger.shouldInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        //处理事件
        mDragger.processTouchEvent(event);
        return true;
    }


}
```
布局文件：
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <me.xfly.viewdragdemo.DragLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:paddingLeft="0dp"
        android:paddingRight="15dp"
        android:paddingTop="0dp"
        android:paddingBottom="15dp"
        android:orientation="vertical">

        <ImageView
            android:id="@+id/iv1"
            android:layout_width="100dp"
            android:layout_height="100dp"
            android:background="#ff0000"
            android:layout_gravity="center_horizontal"
            android:src="@drawable/aaa"></ImageView>

        <ImageView
            android:id="@+id/iv2"
            android:layout_width="100dp"
            android:layout_height="100dp"
            android:layout_gravity="center_horizontal"
            android:src="@drawable/bbb"></ImageView>


    </me.xfly.viewdragdemo.DragLayout>

</LinearLayout>
```
实现效果如下：
![图1](/img/event-4-1.gif)

### ViewDragHelper 的 API
* ViewDragHelper create (ViewGroup forParent, 
                ViewDragHelper.Callback cb) 一个静态的创建方法。
    * 参数一 ViewGroup 自身
    * 参数二 一个回掉，用于实现真正的事件处理
 * shouldInterceptTouchEvent(MotionEvent ev) 判断是否要拦截事件，在 onInterceptTouchEvent 中调用。由 ViewDragHelper 来判断是否要拦截事件。
    * 参数 MotionEvent 代表产生的事件
* processTouchEvent(MotionEvent event)  处理相应事件的方法，在 onTouchEvent 中调用的时候要将返回结果设置为true，否则事件不会被消费。

基本上使用上面这三个 API 就能实现拖拽效果了。其他的如果写后续进阶系列在详细学习。

### ViewDragHelper.Callback的API
* tryCaptureView(View child, int pointerId) 这是一个抽象方法，必须要实现，这个方法返回 true 的时候才生效。
    * 参数一：捕获的 View 也就是你拖动的那个 View。
    * 参数二：手指的 Id。
* onViewDragStateChanged(int state) 当状态改变的时候回掉。
    * state 状态，有三种 STATE_IDLE 闲置状态，STATE_DRAGGING 正在拖动，STATE_SETTLING 放置到某个位置
* onViewPositionChanged(View changedView, int left, int top, int dx, int dy) 被拖动的 View 位置发生改变的时候回掉
    * 参数一 View changedView 被拖动的子 View
    * 参数二 int left 距离左边的距离
    * 参数三 int top 距离顶部的距离
    * 参数四 int dx x 轴上运动的距离
    * 参数五 int dy y 轴上运动的距离
* onViewCaptured(View capturedChild, int activePointerId) 捕获 View 时调用的方法
    * 参数一 View capturedChild 被捕获的 View
    * 参数二 int activePointerId 有效的手指 Id
* onViewReleased(View releasedChild, float xvel, float yvel) View 被释放的时候调用。
    * 参数一 View releasedChild 被释放的子 View
    * 参数二 x 轴的速率
    * 参数三 y 轴的速率
* clampViewPositionVertical(View child, int top, int dy) 计算拖动时垂直位置的方法。
    * View child 被拖动的子 View
    * int top 距离顶部的距离
    * int dy y 轴上的变化量
* clampViewPositionHorizontal(View child, int left, int dx) 计算拖动时水平位置的方法。
    * View child 被拖动的子 View
    * int left 距离左边的距离
    * int dx x 轴上的变化量

比较常用的方法有这么多，但是不需要全部实现，像我们上面的 Demo 只实现了其中的三个方法。

### 源码解析
ViewGroup 处理事件的第一步是判断要不要拦截，当我们用 ViewDragHelper 进行拖动事件处理的时候，是否拦截事件也委托给 ViewDragHelper 处理。这里真正的处理方法是 shouldInterceptTouchEvent 我们看一下源码：
ViewDragHelper#shouldInterceptTouchEvent
```java
public boolean shouldInterceptTouchEvent(MotionEvent ev) {
        //获取action
        final int action = MotionEventCompat.getActionMasked(ev);
        //获取action对应的index
        final int actionIndex = MotionEventCompat.getActionIndex(ev);

        //如果是按下的action则重置一些信息,包括各种事件点的数组
        if (action == MotionEvent.ACTION_DOWN) {
            // Reset things for a new event stream, just in case we didn't get
            // the whole previous stream.
            cancel();
        }
        //初始化mVelocityTracker
        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(ev);

        //根据action来做相应的处理
        switch (action) {
            case MotionEvent.ACTION_DOWN: {
                final float x = ev.getX();
                final float y = ev.getY();
                //获取这个事件对应的pointerId,一般情况下只有一个手指触摸时为0
                //两个手指触摸时第二个手指触摸返回的pointerId为1，以此类推
                final int pointerId = MotionEventCompat.getPointerId(ev, 0);
                //保存点的数据
                //TODO (1)
                saveInitialMotion(x, y, pointerId);
                //获取当前触摸点下最顶层的子View
                //TODO (2)
                final View toCapture = findTopChildUnder((int) x, (int) y);

                //如果toCapture是已经捕获的View,而且正在处于被释放状态
                //那么就重新捕获
                if (toCapture == mCapturedView && mDragState == STATE_SETTLING) {
                    tryCaptureViewForDrag(toCapture, pointerId);
                }

                //如果触摸了边缘,回调callback的onEdgeTouched()方法
                final int edgesTouched = mInitialEdgesTouched[pointerId];
                if ((edgesTouched & mTrackingEdges) != 0) {
                    mCallback.onEdgeTouched(edgesTouched & mTrackingEdges, pointerId);
                }
                break;
            }

            //当又有一个手指触摸时
            case MotionEventCompat.ACTION_POINTER_DOWN: {
                final int pointerId = MotionEventCompat.getPointerId(ev, actionIndex);
                final float x = MotionEventCompat.getX(ev, actionIndex);
                final float y = MotionEventCompat.getY(ev, actionIndex);

                //保存触摸信息
                saveInitialMotion(x, y, pointerId);

                //因为同一时间ViewDragHelper只能操控一个View,所以当有新的手指触摸时
                //只讨论当无触摸发生时,回调边缘触摸的callback
                //或者正在处于释放状态时重新捕获View
                if (mDragState == STATE_IDLE) {
                    final int edgesTouched = mInitialEdgesTouched[pointerId];
                    if ((edgesTouched & mTrackingEdges) != 0) {
                        mCallback.onEdgeTouched(edgesTouched & mTrackingEdges, pointerId);
                    }
                } else if (mDragState == STATE_SETTLING) {
                    // Catch a settling view if possible.
                    final View toCapture = findTopChildUnder((int) x, (int) y);
                    if (toCapture == mCapturedView) {
                        tryCaptureViewForDrag(toCapture, pointerId);
                    }
                }
                break;
            }

            //当手指移动时
            case MotionEvent.ACTION_MOVE: {
                if (mInitialMotionX == null || mInitialMotionY == null) break;

                // First to cross a touch slop over a draggable view wins. Also report edge drags.
                //得到触摸点的数量,并循环处理,只处理第一个发生了拖拽的事件
                final int pointerCount = MotionEventCompat.getPointerCount(ev);
                for (int i = 0; i < pointerCount; i++) {
                    final int pointerId = MotionEventCompat.getPointerId(ev, i);
                    final float x = MotionEventCompat.getX(ev, i);
                    final float y = MotionEventCompat.getY(ev, i);
                    //获得拖拽偏移量
                    final float dx = x - mInitialMotionX[pointerId];
                    final float dy = y - mInitialMotionY[pointerId];
                    //获取当前触摸点下最顶层的子View
                    final View toCapture = findTopChildUnder((int) x, (int) y);
                    //如果找到了最顶层View,并且产生了拖动(checkTouchSlop()返回true)
                    //TODO (3)
                    final boolean pastSlop = toCapture != null && checkTouchSlop(toCapture, dx, dy);
                    if (pastSlop) {
                        //根据callback的四个方法getView[Horizontal|Vertical]DragRange和
                        //clampViewPosition[Horizontal|Vertical]来检查是否可以拖动
                        final int oldLeft = toCapture.getLeft();
                        final int targetLeft = oldLeft + (int) dx;
                        final int newLeft = mCallback.clampViewPositionHorizontal(toCapture,
                                targetLeft, (int) dx);
                        final int oldTop = toCapture.getTop();
                        final int targetTop = oldTop + (int) dy;
                        final int newTop = mCallback.clampViewPositionVertical(toCapture, targetTop,
                                (int) dy);
                        final int horizontalDragRange = mCallback.getViewHorizontalDragRange(
                                toCapture);
                        final int verticalDragRange = mCallback.getViewVerticalDragRange(toCapture);
                        //如果都不允许移动则跳出循环
                        if ((horizontalDragRange == 0 || horizontalDragRange > 0
                                && newLeft == oldLeft) && (verticalDragRange == 0
                                || verticalDragRange > 0 && newTop == oldTop)) {
                            break;
                        }
                    }
                    //记录并回调是否有边缘触摸
                    reportNewEdgeDrags(dx, dy, pointerId);
                    if (mDragState == STATE_DRAGGING) {
                        // Callback might have started an edge drag
                        break;
                    }
                    //如果产生了拖动则调用tryCaptureViewForDrag()
                    //TODO (4)
                    if (pastSlop && tryCaptureViewForDrag(toCapture, pointerId)) {
                        break;
                    }
                }
                //保存触摸点的信息
                saveLastMotion(ev);
                break;
            }

            //当有一个手指抬起时,清除这个手指的触摸数据
            case MotionEventCompat.ACTION_POINTER_UP: {
                final int pointerId = MotionEventCompat.getPointerId(ev, actionIndex);
                clearMotionHistory(pointerId);
                break;
            }

            //清除所有触摸数据
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL: {
                cancel();
                break;
            }
        }

        //如果mDragState等于正在拖拽则返回true
        return mDragState == STATE_DRAGGING;
    }
```
上面就是整个shouldInterceptTouchEvent()的实现,上面的注释也足够清楚了,我们这里就先不分析某一种触摸事件。

ViewDragHepler#processTouchEvent
```java
 public void processTouchEvent(MotionEvent ev) {
        final int action = MotionEventCompat.getActionMasked(ev);
        final int actionIndex = MotionEventCompat.getActionIndex(ev);

        ...（省去部分代码）
        switch (action) {
            case MotionEvent.ACTION_DOWN: {
                ...（省去部分代码）
                break;
            }

            case MotionEventCompat.ACTION_POINTER_DOWN: {
                ...（省去部分代码）
                break;
            }

            case MotionEvent.ACTION_MOVE: {
                //如果现在已经是拖拽状态
                if (mDragState == STATE_DRAGGING) {
                    final int index = MotionEventCompat.findPointerIndex(ev, mActivePointerId);
                    final float x = MotionEventCompat.getX(ev, index);
                    final float y = MotionEventCompat.getY(ev, index);
                    final int idx = (int) (x - mLastMotionX[mActivePointerId]);
                    final int idy = (int) (y - mLastMotionY[mActivePointerId]);

                    //拖拽至指定位置
                    //TODO (5)
                    dragTo(mCapturedView.getLeft() + idx, mCapturedView.getTop() + idy, idx, idy);

                    saveLastMotion(ev);
                } else {
                    // Check to see if any pointer is now over a draggable view.
                    //如果还不是拖拽状态,就检测是否经过了一个View
                    final int pointerCount = MotionEventCompat.getPointerCount(ev);
                    for (int i = 0; i < pointerCount; i++) {
                        final int pointerId = MotionEventCompat.getPointerId(ev, i);
                        final float x = MotionEventCompat.getX(ev, i);
                        final float y = MotionEventCompat.getY(ev, i);
                        final float dx = x - mInitialMotionX[pointerId];
                        final float dy = y - mInitialMotionY[pointerId];

                        reportNewEdgeDrags(dx, dy, pointerId);
                        if (mDragState == STATE_DRAGGING) {
                            // Callback might have started an edge drag.
                            break;
                        }

                        final View toCapture = findTopChildUnder((int) x, (int) y);
                        if (checkTouchSlop(toCapture, dx, dy) &&
                                tryCaptureViewForDrag(toCapture, pointerId)) {
                            break;
                        }
                    }
                    saveLastMotion(ev);
                }
                break;
            }
            //当多个手指中的一个手机松开时
            case MotionEventCompat.ACTION_POINTER_UP: {
                final int pointerId = MotionEventCompat.getPointerId(ev, actionIndex);
                //如果当前点正在被拖拽,则再剩余还在触摸的点钟寻找是否正在View上
                if (mDragState == STATE_DRAGGING && pointerId == mActivePointerId) {
                    // Try to find another pointer that's still holding on to the captured view.
                    int newActivePointer = INVALID_POINTER;
                    final int pointerCount = MotionEventCompat.getPointerCount(ev);
                    for (int i = 0; i < pointerCount; i++) {
                        final int id = MotionEventCompat.getPointerId(ev, i);
                        if (id == mActivePointerId) {
                            // This one's going away, skip.
                            continue;
                        }

                        final float x = MotionEventCompat.getX(ev, i);
                        final float y = MotionEventCompat.getY(ev, i);
                        if (findTopChildUnder((int) x, (int) y) == mCapturedView &&
                                tryCaptureViewForDrag(mCapturedView, id)) {
                            newActivePointer = mActivePointerId;
                            break;
                        }
                    }

                    if (newActivePointer == INVALID_POINTER) {
                        // We didn't find another pointer still touching the view, release it.
                        //如果没找到则释放View
                        //TODO (6)
                        releaseViewForPointerUp();
                    }
                }
                clearMotionHistory(pointerId);
                break;
            }

            case MotionEvent.ACTION_UP: {
                //如果是拖拽状态的释放则调用
                //releaseViewForPointerUp()
                if (mDragState == STATE_DRAGGING) {
                    releaseViewForPointerUp();
                }
                cancel();
                break;
            }

            case MotionEvent.ACTION_CANCEL: {
                if (mDragState == STATE_DRAGGING) {
                    dispatchViewReleased(0, 0);
                }
                cancel();
                break;
            }
        }
    }
```
以上是 ViewDragHelper 中的两个关键方法，他们也调用了其他的一些方法，我们来看一下比较关键的几个：

ViewDragHelper#findTopChildUnder
```java
 public View findTopChildUnder(int x, int y) {
        final int childCount = mParentView.getChildCount();
        for (int i = childCount - 1; i >= 0; i--) {
            final View child = mParentView.getChildAt(mCallback.getOrderedChildIndex(i));
            if (x >= child.getLeft() && x < child.getRight() &&
                    y >= child.getTop() && y < child.getBottom()) {
                return child;
            }
        }
        return null;
    }
```
代码很简单，就是根据 x 和 y 坐标找到目标 View ，注意这里回掉了 callback 中的 getOrderedChildIndex(int index) 方法，它的默认实现是返回当前的 index，如果我们要更改目标 index 可以在 callback 中覆写这个方法。

ViewDragHelper#dragTo
```java
 private void dragTo(int left, int top, int dx, int dy) {
        int clampedX = left;
        int clampedY = top;
        final int oldLeft = mCapturedView.getLeft();
        final int oldTop = mCapturedView.getTop();
        if (dx != 0) {
            //回调callback来决定View最终被拖拽的x方向上的偏移量
            clampedX = mCallback.clampViewPositionHorizontal(mCapturedView, left, dx);
            //移动View
            ViewCompat.offsetLeftAndRight(mCapturedView, clampedX - oldLeft);
        }
        if (dy != 0) {
            //回调callback来决定View最终被拖拽的y方向上的偏移量
            clampedY = mCallback.clampViewPositionVertical(mCapturedView, top, dy);
            //移动View
            ViewCompat.offsetTopAndBottom(mCapturedView, clampedY - oldTop);
        }

        if (dx != 0 || dy != 0) {
            final int clampedDx = clampedX - oldLeft;
            final int clampedDy = clampedY - oldTop;
            //回调callback
            mCallback.onViewPositionChanged(mCapturedView, clampedX, clampedY,
                    clampedDx, clampedDy);
        }
    }
```
因为dragTo()方法是在processTouchEvent()中的MotionEvent.ACTION_MOVE case被调用所以当程序运行到这里时View就会不断的被拖动了。如果一旦手指释放则最终会调用releaseViewForPointerUp()方法

ViewDragHelper#releaseViewForPointerUp
```java
 private void releaseViewForPointerUp() {
        //计算出当前x和y方向上的加速度
        mVelocityTracker.computeCurrentVelocity(1000, mMaxVelocity);
        final float xvel = clampMag(
                VelocityTrackerCompat.getXVelocity(mVelocityTracker, mActivePointerId),
                mMinVelocity, mMaxVelocity);
        final float yvel = clampMag(
                VelocityTrackerCompat.getYVelocity(mVelocityTracker, mActivePointerId),
                mMinVelocity, mMaxVelocity);
        dispatchViewReleased(xvel, yvel);
    }
```
计算完加速度之后就调用了 dispatchViewReleased():

ViewDragHelper#dispatchViewReleased
```java
  private void dispatchViewReleased(float xvel, float yvel) {
        //设定当前正处于释放阶段
        mReleaseInProgress = true;
        //回调callback的onViewReleased()方法
        mCallback.onViewReleased(mCapturedView, xvel, yvel);
        mReleaseInProgress = false;

        //设定状态
        if (mDragState == STATE_DRAGGING) {
            // onViewReleased didn't call a method that would have changed this. Go idle.
            //如果onViewReleased()中没有调用任何方法,则状态设定为STATE_IDLE
            setDragState(STATE_IDLE);
        }
    }
```
所以最后释放后的处理交给了callback中的onViewReleased()方法,如果我们什么都不做,那么这个被拖拽的View就是停止在当前位置,或者我们可以调用ViewDragHelper提供给我们的这几个方法:

* settleCapturedViewAt(int finalLeft, int finalTop)
以松手前的滑动速度为初速动，让捕获到的View自动滚动到指定位置。只能在Callback的onViewReleased()中调用。
* flingCapturedView(int minLeft, int minTop, int maxLeft, int maxTop)
以松手前的滑动速度为初速动，让捕获到的View在指定范围内fling。只能在Callback的onViewReleased()中调用。
* smoothSlideViewTo(View child, int finalLeft, int finalTop)
指定某个View自动滚动到指定的位置，初速度为0，可在任何地方调用。
