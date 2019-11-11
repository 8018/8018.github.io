---
layout:     post
title:      Android 控件-6 ViewRootImpl - 4 performTraversals 之布局
date:       2015-07-05
author:     xflyme
header-img: img/post-bg-2015-07-05.png
catalog: true
tags:
    - Android
    - 控件
    - 源码分析
---


### 前言
前两节学习了 ViewRootImpl 中的测量和窗口布局，接下来我们看 View 定制的三剑客之一布局。

### 布局控件树
经过前面的测量，控件树中的控件的尺寸已经确定，而父控件对于子控件的位置也已了解。布局阶段就是把测量结果实现，即把测量结果转化为控件实际的位置和尺寸。

控件的实际位置有四个成员变量表示：mLeft、mTop、mRight、mBottom，因此控件树的布局过程就是根据测量结果为每一个控件设置这四个成员变量的过程。

> mLeft、mTop、mRight、mBottom 这些坐标是相对于父控件左上角来说的。

```java
public void performTraversals(){
      ...
      //条件 layoutRequested
      final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
        boolean triggerGlobalLayoutListener = didLayout
                || mAttachInfo.mRecomputeGlobalAttributes;
        if (didLayout) {
            //通过 performLayout 进行控件树布局
            performLayout(lp, mWidth, mHeight);

            // By this point all views have been sized and positioned
            // We can compute the transparent area
            //如有必要计算窗口的透明区域并将其设置给 WMS
            if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
                // start out transparent
               
                host.getLocationInWindow(mTmpLocation);
                //将透明区域初始化为整个 mView 的区域
                mTransparentRegion.set(mTmpLocation[0], mTmpLocation[1],
                        mTmpLocation[0] + host.mRight - host.mLeft,
                        mTmpLocation[1] + host.mBottom - host.mTop);

//通过 gatherTransparentRegion 遍历控件树中的每一个控件，倘若控件有内容需要绘制，将其区域从 mTransparentRegion 中移除
host.gatherTransparentRegion(mTransparentRegion);
                if (mTranslator != null) {
                    mTranslator.translateRegionInWindowToScreen(mTransparentRegion);
                }
                //将透明区域设置给 WMS
                if (!mTransparentRegion.equals(mPreviousTransparentRegion)) {
                    mPreviousTransparentRegion.set(mTransparentRegion);
                    mFullRedrawNeeded = true;
                    // reconfigure window manager
                    try {
                        mWindowSession.setTransparentRegion(mWindow, mTransparentRegion);
                    } catch (RemoteException e) {
                    }
                }
            }

            if (DBG) {
                System.out.println("======================================");
                System.out.println("performTraversals -- after setFrame");
                host.debug();
            }
        }
        ...
}
```

布局控件树的条件不像布局窗口那么严格，只要 layoutRequested 为 true，即 requestLayout() 被调用过即可。

requestLayout() 的用途相当于控件树中的某一个控件需要改变自身尺寸时，向 ViewRootImpl 申请一次新的测量、布局以及绘制。

布局控件树阶段主要做了两件事：
* 进行控件树布局
* 设置窗口透明区域

#### 一 控件树布局

ViewRootImpl#performLayout
```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
       ...

        final View host = mView;
        

        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "layout");
        try {
        //调用 mView.layout() 启动布局流程
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

            mInLayout = false;
            int numViewsRequestingLayout = mLayoutRequesters.size();
            if (numViewsRequestingLayout > 0) {
                // requestLayout() was called during layout.
                // If no layout-request flags are set on the requesting views, there is no problem.
                // If some requests are still pending, then we need to clear those flags and do
                // a full request/measure/layout pass to handle this situation.
                ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                        false);
                if (validLayoutRequesters != null) {
                    // Set this flag to indicate that any further requests are happening during
                    // the second pass, which may result in posting those requests to the next
                    // frame instead
                    mHandlingLayoutInLayoutRequest = true;

                    // Process fresh layout requests, then measure and layout
                    int numValidRequests = validLayoutRequesters.size();
                    for (int i = 0; i < numValidRequests; ++i) {
                        final View view = validLayoutRequesters.get(i);
                        Log.w("View", "requestLayout() improperly called by " + view +
                                " during layout: running second layout pass");
                        view.requestLayout();
                    }
                    measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight);
                    mInLayout = true;
                    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

                    mHandlingLayoutInLayoutRequest = false;

                    // Check the valid requests again, this time without checking/clearing the
                    // layout flags, since requests happening during the second pass get noop'd
                    validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                    if (validLayoutRequesters != null) {
                        final ArrayList<View> finalRequesters = validLayoutRequesters;
                        // Post second-pass requests to the next frame
                        getRunQueue().post(new Runnable() {
                            @Override
                            public void run() {
                                int numValidRequests = finalRequesters.size();
                                for (int i = 0; i < numValidRequests; ++i) {
                                    final View view = finalRequesters.get(i);
                                    Log.w("View", "requestLayout() improperly called by " + view +
                                            " during second layout pass: posting in next frame");
                                    view.requestLayout();
                                }
                            }
                        });
                    }
                }

            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        mInLayout = false;
    }
```

接着往下看：
View#layout
```java
public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }
        //保存原始坐标
        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;
        
        //设置 Frame
        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        //执行 onLayout
        //如果是 ViewGroup 其中会依次调用子控件的 layout 方法
            onLayout(changed, l, t, r, b);

            if (shouldDrawRoundScrollbar()) {
                if(mRoundScrollbarRenderer == null) {
                    mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
                }
            } else {
                mRoundScrollbarRenderer = null;
            }

            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            //通知对此控件布局变化感兴趣的监听者
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

        if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
            mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
            notifyEnterOrExitForAutoFillIfNeeded(true);
        }
    }
```
这一阶段主要做了三件事：
* 通过 setFrame 设置布局的四个坐标
* 调用 onLayout() 方法，如果此控件是一个 ViewGroup 会在 onLayout() 中依次调用子控件的 layout() 。
* 通知对此控件布局变化感兴趣的监听者。

> 注意测量过程是后根遍历，子控件的测量影响父控件的测量结果。
> 布局过程是先根遍历，因为父控件的布局结果影响子控件的布局结果。

#### 二 窗口透明区域

所谓透明区域是指 Surface 上的一块特定区域，在 SurfaceFlinger 进行混层时，Surface 上的这块区域将被忽略，就好像在 Surface 上切了一个洞一般。

透明区域的计算由 View.gatherTransparentRegion() 方法完成。透明区域的计算采用了挖洞法，默认整个窗口都是透明区域。然后遍历整个控件树，在 gatherTransparentRegion() 遍历到一个控件时，如果这个控件有内容需要绘制，则将其所在的区域从当前透明区域中移除，当遍历完成后，剩余的区域就是最终的透明区域。
