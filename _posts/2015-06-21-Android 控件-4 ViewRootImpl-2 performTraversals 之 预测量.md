---
layout:     post
title:      Android 控件-4 ViewRootImpl-2 performTraversals 之 预测量
date:       2015-06-21
author:     xflyme
header-img: img/post-bg-2015-06-21.jpg
catalog: true
tags:
    - Android
    - 控件
    - 源码分析
---


### 简介
performTraversals() 是一个保罗万象的方法。ViewRootImpl 中接收的各种变化，如来自 WMS 的窗口属性的变化，来自控件树的尺寸变化以及重绘请求等都会引发 performTraversals() 的调用。View 及其子类的 onMeasure()、onLayout()、onDraw() 都是在 performTraversals() 执行过程中直接或间接的引发。可以说是 performTraversals() 驱动着整个控件树有条不紊的运行，这几个小节我们重点学习下 performTraversals()，本节由 measure 开始。

### 源码

整个 performTraversals() ，有 800 多行，我们去掉不重要的代码，重点代码加上注释，一步步看一下。


```java
private void performTraversals() {
    // 下面有很多次用到 mView 将它存储到局部变量里，提高访问效率
    final View host = mView;

       ...

    // 这两个变量是 mView SPEC_SIZE 的候选
    int desiredWindowWidth;
    int desiredWindowHeight;

    ...
    
    //mWinFrame 表示窗口的最新尺寸
    Rect frame = mWinFrame;
    if (mFirst) {
    //表示 第一次，此时窗口刚被添加到 WMS ，窗口尚未进行 relayout ，因此 mWinFrame 中没有存储窗口的有效尺寸
        mFullRedrawNeeded = true;
        mLayoutRequested = true;

        final Configuration config = mContext.getResources().getConfiguration();
        if (shouldUseDisplaySize(lp)) {
            // NOTE -- system code, won't try to do compat mode.
            Point size = new Point();
            //屏幕的真实尺寸，不包含任何 DecorView
            mDisplay.getRealSize(size);
            desiredWindowWidth = size.x;
            desiredWindowHeight = size.y;
        } else {
             // 第一次 使用 config 配置的宽高
             desiredWindowWidth = dipToPx(config.screenWidthDp);
             desiredWindowHeight = dipToPx(config.screenHeightDp);
        }

        //由于是第一次「遍历」，填充 mAttachInfo 的一些字段
    } else {
    // 非第一次的情况下，采用窗口的最新尺寸作为 SPEC_SIZE 候选
        desiredWindowWidth = frame.width();
        desiredWindowHeight = frame.height();
        
        //如果窗口尺寸和控件树中的尺寸不同，说明 WMS 单方面改变了窗口尺寸，将会导致一下三个结果
        if (desiredWindowWidth != mWidth || desiredWindowHeight != mHeight) {
            if (DEBUG_ORIENTATION) Log.v(mTag, "View " + host + " resized to: " + frame);
            //需要重新进行完整的重绘以适应新的窗口尺寸
            mFullRedrawNeeded = true;
            //需要对控件树进行重新布局
            mLayoutRequested = true;
            //控件树有可能拒绝接受新的窗口尺寸，比如在随后的预测量中给出不同于窗口尺寸的测量结果。产生这种情况就需要在窗口布局阶段尝试设置新的窗口尺寸。
            windowSizeMayChange = true;
        }
    }

   ...

    // Execute enqueued actions on every traversal in case a detached view enqueued an action
    getRunQueue().executeActions(mAttachInfo.mHandler);

    boolean insetsChanged = false;

    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    // 仅当 layoutRequested 为 true 时才进行预测量，layoutRequested 为true 表示进行「遍历」之前 requestLayout（） 方法被调用过，
    if (layoutRequested) {

        final Resources res = mView.getContext().getResources();

        if (mFirst) {
            // make sure touch mode code executes by setting cached value
            // to opposite of the added touch mode.
            mAttachInfo.mInTouchMode = !mAddedTouchMode;
            ensureTouchModeLocally(mAddedTouchMode);
        } else {
            if (!mPendingOverscanInsets.equals(mAttachInfo.mOverscanInsets)) {
                insetsChanged = true;
            }
            if (!mPendingContentInsets.equals(mAttachInfo.mContentInsets)) {
                insetsChanged = true;
            }
            if (!mPendingStableInsets.equals(mAttachInfo.mStableInsets)) {
                insetsChanged = true;
            }
            if (!mPendingDisplayCutout.equals(mAttachInfo.mDisplayCutout)) {
                insetsChanged = true;
            }
            if (!mPendingVisibleInsets.equals(mAttachInfo.mVisibleInsets)) {
                mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
                if (DEBUG_LAYOUT) Log.v(mTag, "Visible insets changing to: "
                        + mAttachInfo.mVisibleInsets);
            }
            if (!mPendingOutsets.equals(mAttachInfo.mOutsets)) {
                insetsChanged = true;
            }
            if (mPendingAlwaysConsumeNavBar != mAttachInfo.mAlwaysConsumeNavBar) {
                insetsChanged = true;
            }
            // 当窗口的 width 或 height 被设置为 WRAP_CONTENT 表示这是一个悬浮窗口，此时会对 desiredWindowWidth/desiredWindowHeight 进行调整
            if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT
                    || lp.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
                windowSizeMayChange = true;

                if (shouldUseDisplaySize(lp)) {
                    // NOTE -- system code, won't try to do compat mode.
                    Point size = new Point();
                    mDisplay.getRealSize(size);
                    desiredWindowWidth = size.x;
                    desiredWindowHeight = size.y;
                } else {
                    Configuration config = res.getConfiguration();
                    desiredWindowWidth = dipToPx(config.screenWidthDp);
                    desiredWindowHeight = dipToPx(config.screenHeightDp);
                }
            }
        }

        // 通过 measureHierarchy 进行预测量
        windowSizeMayChange |= measureHierarchy(host, lp, res,
                desiredWindowWidth, desiredWindowHeight);
    }

    //其他阶段的处理
    ...
}
```
> 上面不同情形下，为 desiredWindowWidth/desiredWindowHeight 选择了不同的候选尺寸，这些尺寸有什么不同？

#### 协商测量
measureHierarchy() 用于测量整个控件树，下面看一下它的源码：

ViewRootImpl#measureHierarchy()
```java
 private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
        //合成用于描述宽度的 MeasureSpec
        int childWidthMeasureSpec;
        //合成用于描述高度的 MeasureSpec
        int childHeightMeasureSpec;
        //测量结果可能导致窗口变化
        boolean windowSizeMayChange = false;
        //测量结果能满足控件树充分显示内容的要求
        boolean goodMeasure = false;
        
        // 只有在 LayoutParams.width 被设置为 WRAP_CONTENT 才会发生协商测量
        // 比如一个弹窗，我们并不希望弹窗占满整个屏幕去显示一行文字
        if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
            
            final DisplayMetrics packageMetrics = res.getDisplayMetrics();
            //第一次协商， measureHierarchy 使用它期望的宽度进行测量，这种宽度限制是一种系统预先配置的宽度，在/frameworks/base/core/res/res/values/confit.xml 中可以找到它  
                   res.getValue(com.android.internal.R.dimen.config_prefDialogWidth, mTmpValue, true);
            int baseSize = 0;
            //宽度存储在 baseSize 中
            if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {
                baseSize = (int)mTmpValue.getDimension(packageMetrics);
            }
            if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": baseSize=" + baseSize
                    + ", desiredWindowWidth=" + desiredWindowWidth);
            if (baseSize != 0 && desiredWindowWidth > baseSize) {
            //baseSize 比 desiredWindowWidth 小，先使用 baseSize 合成 MeasureSpec 看能否满足控件要求
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
                //第一次测量，由 performMeasure 完成
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": measured ("
                        + host.getMeasuredWidth() + "," + host.getMeasuredHeight()
                        + ") from width spec: " + MeasureSpec.toString(childWidthMeasureSpec)
                        + " and height spec: " + MeasureSpec.toString(childHeightMeasureSpec));
    //测量结果可以由 getMeasuredWidthAndState 获得，如果控件树对这个测量结果不满意，会在返回值中添加 MEASURED_STATE_TOO_SMALL
                if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                //控件树对测量结果满意
                    goodMeasure = true;
                } else {
                    //控件树对测量结果不满意
                    //适当放宽 baseSize
                    baseSize = (baseSize+desiredWindowWidth)/2;
                    if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": next baseSize="
                            + baseSize);
                    childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                    //第二次测量
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                    if (DEBUG_DIALOG) Log.v(mTag, "Window " + mView + ": measured ("
                            + host.getMeasuredWidth() + "," + host.getMeasuredHeight() + ")");
                    //再次检查结果能否满足控件树的要求
                    if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                        if (DEBUG_DIALOG) Log.v(mTag, "Good!");
                        goodMeasure = true;
                    }
                }
            }
        }
    //最终测量， measureHierarchy 放弃所有限制做最终测量
    //这一次不在检查控件树是否满意，因为即使不满意，也没有更多的空间了
    
        if (!goodMeasure) {
            childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            //最后如果测量结果和 ViewRootImpl 中的窗口尺寸不一致，表示随后可能会进行窗口尺寸的调整。
            if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
                windowSizeMayChange = true;
            }
        }

        return windowSizeMayChange;
    }
```
> LayoutParams.Width 被设置为 WRAP_CONTENT 时存在协商过程，非 WRAP_CONTENT 不存在协商。
> 存在协商的情况下 measureHierarchy 最多可进行两次让步，即在最不利的情况下，ViewRootImpl 中的一次遍历，控件树需要进行三次测量。

![图1](/img/android-view-4-1.png)

### performMeasure()

接下来通过 performMeasure() 看控件树如何测量，其实 performMeasure() 代码比较简单，它直接调用 mView.measure() 将 measureHierarchy 生成的 MeasureSpec 传进去。

ViewRootImpl#performMeasure
```java

    private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        if (mView == null) {
            return;
        }
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```

View#measure

```java
  public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        // Suppress sign extension for the low bytes
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);
        
        // MeasureSpec 发生变化或强制重新布局时才会进行测量
        //强制重新布局是指一个子控件内容发生变化，在这种情况下，子控件的父控件（父控件的父控件）提供的 MeasureSpec 必定与上次相同，导致父控件的 measure()方法无法得到执行，进而导致子控件无法测量其尺寸和布局。
        //怎么解决？子控件内容发生变化时，依次调用父控件的 requestLayout()，这个方法将会在 mPrivateFlags 加入 PFLAG_FORCE_LAYOUT 从而使父控件的 measure() 方法得以执行。
        
        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;

        // Optimize layout by avoiding an extra EXACTLY pass when the view is
        // already measured as the correct size. In API 23 and below, this
        // extra pass is required to make LinearLayout re-distribute weight.
        final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
                || heightMeasureSpec != mOldHeightMeasureSpec;
        final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
                && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
        final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
                && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
        final boolean needsLayout = specChanged
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);

        if (forceLayout || needsLayout) {
            // first clears the measured dimension flag
            //准备，从 mPrivateFlags 将 PFLAG_MEASURED_DIMENSION_SET 标志去除
            //该标志用于检查是否已经调用 setMeasuredDimension 存储测量结果
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded();

            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                //对本控件进行测量
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            // flag not set, setMeasuredDimension() was not invoked, we raise
            // an exception to warn the developer
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }

            //将 PFLAG_LAYOUT_REQUIRED 标志加入 mPrivateFlags
            //有了这个标志，后面的布局才得以放行
            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }
        //记录旧的 MeasureSpec 以便后续检查 measure 是否有必要进行。
        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }

```

从以上代码中可以看出 View.measure() 没有实现任何测量算法，它只是一个调度方法，它的作用是：
> 触发 onMeasure
> 对 onMeasure 的结果进行检查，并将之通过 setMeasuredDimension 进行保存。

继续，看一下 setMeasuredDimension：

View#setMeasuredDimension
```java
    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }


 private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;

        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
```
很简单，不在一一解释了，不过有一点需要注意。
> 测量结果不仅仅是一个尺寸，而是一个状态与尺寸的复合型变量，0到30位表示测量的结果尺寸，31位和32位表示控件对测量结果是否满意。

#### 确定是否需要改变窗口尺寸
接下来回到 performTraversals 方法，在 measureHierarchy 执行完毕之后，ViewRootImpl 了解了控件树所需要的空间。便可以确定是否需要改变窗口尺寸以满足控件树的要求。上面的代码中多处设置 windowSizeMayChange 为 true，windowSizeMayChange 仅表示有可能需要调整窗口尺寸，而接下来的这段代码用来确定是否需要改变窗口尺寸。

```java
  private void performTraversals() {
        
        ...

        if (layoutRequested) {
            // Clear this now, so that if anything requests a layout in the
            // rest of this function we will catch it and re-run a full
            // layout pass.
            mLayoutRequested = false;
        }

        boolean windowShouldResize = layoutRequested && windowSizeMayChange
            && ((mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight())
                || (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT &&
                        frame.width() < desiredWindowWidth && frame.width() != mWidth)
                || (lp.height == ViewGroup.LayoutParams.WRAP_CONTENT &&
                        frame.height() < desiredWindowHeight && frame.height() != mHeight));
       
       ...
    }
```
 windowShouldResize 的条件看起来比较复杂，我们拆开来看一下：
 * layoutRequested 为 true ，即 requestLayout() 被调用过，当控件内容发生变化需要调整尺寸时，requestLayout() 回溯到 ViewRootImpl 从而引发 performTraversals。这是一个必要条件是因为有可能控件只需要重绘，不需要重新布局。
 比如通过 invalidate() 回溯到根部，此时通过 scheduleTraversals() 触发 performTraversals() 而不是通过 requestLayout() 触发，此时只重绘，尺寸不发生变化。
* windowSizeMayChange 为 true，这表示控件树测量结果和窗口尺寸不一致。
* 再满足以上两条条件之后，一下条件满足一个即可：
    * 测量结果和 ViewRootImpl 中保存的当前尺寸不一致。
    * 悬浮窗口的测量结果与窗口的最新尺寸有差异。

### 总结
* 预测量的触发我们上一节已经分析过了，目的是确保在「 window 被添加到 WindowManager」之前触发一次遍历，以便后续接收各种事件。
* 第一次测量的时候 mWinFrame 还未被赋值，因此预测量使用的 SPEC_SIZE 分量是从 Configuration 配置文件中取出的。
* 如果 lp.width 为 WRAP_CONTENT measureHierarchy 的时候会进行协商测量，逐步放宽测量条件看是否能满足控件树要求。
* setMeasuredDimension 保存的结果不仅仅是一个数值，它是一个复合变量，其 31 和 32 位用来表示测量结果能否满足控件要求，如果不能满足还需要重新测量。
