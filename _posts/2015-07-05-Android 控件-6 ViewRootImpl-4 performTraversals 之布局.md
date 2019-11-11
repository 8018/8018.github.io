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


### 简介
上一节学习了 performTraversals 中的预测量，预测量之后是窗口布局和最终测量。窗口布局阶段以 relatyoutWindow() 为核心，并根据布局结果进行相应处理，当布局结果使得窗口尺寸发生变化时，最终测量将会被执行。

### 布局窗口的条件
窗口布局阶段的开销很大，因此必须限制窗口布局阶段的执行，倘若不需要进行窗口布局，即 WMS 不会在预测量之后修改窗口尺寸，最终测量也不再需要。

参考如下代码：

```java
 private void performTraversals() {
        //预测量阶段
       ...
         if (mFirst || windowShouldResize || insetsChanged ||
                viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
           ... // 布局窗口与最终测量
        } else {
            // Not the first pass and no window/insets/visibility change but the window
            // may have moved and we need check that and if so to update the left and right
            // in the attach info. We translate only the window frame since on window move
            // the window manager tells us only for the new frame but the insets are the
            // same and we do not want to translate them more than once.
            //不符合布局窗口的条件，窗口的尺寸不需要调整，有可能只是窗口的位置发生变化，仅需将窗口的最新位置保存到 mAttachInfo
            maybeHandleWindowMove(frame);
        }
        
        ...
        //布局控件树与绘制
    }
    
    
    
    
    private void maybeHandleWindowMove(Rect frame) {

        // TODO: Well, we are checking whether the frame has changed similarly
        // to how this is done for the insets. This is however incorrect since
        // the insets and the frame are translated. For example, the old frame
        // was (1, 1 - 1, 1) and was translated to say (2, 2 - 2, 2), now the new
        // reported frame is (2, 2 - 2, 2) which implies no change but this is not
        // true since we are comparing a not translated value to a translated one.
        // This scenario is rare but we may want to fix that.

        final boolean windowMoved = mAttachInfo.mWindowLeft != frame.left
                || mAttachInfo.mWindowTop != frame.top;
        if (windowMoved) {
            if (mTranslator != null) {
                mTranslator.translateRectInScreenToAppWinFrame(frame);
            }
            mAttachInfo.mWindowLeft = frame.left;
            mAttachInfo.mWindowTop = frame.top;
        }
        if (windowMoved || mAttachInfo.mNeedsUpdateLightCenter) {
            // Update the light position for the new offsets.
            if (mAttachInfo.mThreadedRenderer != null) {
                mAttachInfo.mThreadedRenderer.setLightCenter(mAttachInfo);
            }
            mAttachInfo.mNeedsUpdateLightCenter = false;
        }
    }
```

进入窗口布局有一个条件，以下满足一个即可进入窗口布局：
* mFirst 表示第一次，此时窗口仅仅是添加到 WMS，尚未进行窗口布局也没有有效的 Surface，因此必须进行窗口布局。
* windowShouldResize 控件树的测量结果与窗口尺寸有差异，需要通过布局控件树向 WMS 提出修改窗口尺寸以满足控件树的要求。
* insetChanged 表示 WMS 单方便改变了 ContentInsets，这种一般发生在 SystemUI 或输入法发生可见性变化。严格来说，这种情况下不需要重新布局，只需要一段动画，让控件树过渡到新的 ContentInsets 下。
* params ！= null 进入 performTraversals() 时，params 被置为 null，此时说明 params 发生了变化，需要将新的 params 更新到 WMS 中，使其对窗口依照新的属性进行重新布局。

#### 布局窗口的准备工作

```java
private void performTraversals() {
        //预测量阶段
       ...
         if (mFirst || windowShouldResize || insetsChanged ||
                viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
                //是否拥有有效的 Surface
            boolean hadSurface = mSurface.isValid();
            //记录之前 Surface id
            final int surfaceGenerationId = mSurface.getGenerationId();
            //通过relayoutWindow 布局窗口
            relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
            ...
        } else {
            ...
        }
        
        ...
        //布局控件树与绘制
    }
    


```

可以看出布局窗口之前的主要操作是保存布局前的状态，Surface 的两个状态只是准备工作的一部分，有些状态已经准备好了如 mWidth/mHeight 等。

### 布局窗口
ViewRootImpl 使用 relayoutWindow() 方法进行窗口布局，代码如下：

ViewRootImpl#relayoutWindow
```java
private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {

        float appScale = mAttachInfo.mApplicationScale;
        ...
        int relayoutResult = mWindowSession.relayout(
                mWindow, mSeq, params,
                (int) (mView.getMeasuredWidth() * appScale + 0.5f),
                (int) (mView.getMeasuredHeight() * appScale + 0.5f),
                viewVisibility, insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0,
                mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
                mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame,
                mPendingMergedConfiguration, mSurface);

        mPendingAlwaysConsumeNavBar =
                (relayoutResult & WindowManagerGlobal.RELAYOUT_RES_CONSUME_ALWAYS_NAV_BAR) != 0;

       ...
        return relayoutResult;
    }
```
由以上代码看出，这只是 WMS.relayoutWindow 的包装方法。其中它将窗口的 LayoutParams、预测量结果以及 mView 的可见性作为输入，并将 mWinFrame、mPendingContentInsets、mSurface 作为输出。

### 布局窗口后的处理 —— insets
完成窗口布局之后，回到 performTraversals()，对布局结果所导致的变化进行处理，首先是 ContentInsets 和 VisibleInsets:
```java
private void performTraversals() {
                ...
    
                final boolean overscanInsetsChanged = !mPendingOverscanInsets.equals(
                        mAttachInfo.mOverscanInsets);
                contentInsetsChanged = !mPendingContentInsets.equals(
                        mAttachInfo.mContentInsets);
                final boolean visibleInsetsChanged = !mPendingVisibleInsets.equals(
                        mAttachInfo.mVisibleInsets);
                final boolean stableInsetsChanged = !mPendingStableInsets.equals(
                        mAttachInfo.mStableInsets);
                final boolean outsetsChanged = !mPendingOutsets.equals(mAttachInfo.mOutsets);
                final boolean surfaceSizeChanged = (relayoutResult
                        & WindowManagerGlobal.RELAYOUT_RES_SURFACE_RESIZED) != 0;
                final boolean alwaysConsumeNavBarChanged =
                        mPendingAlwaysConsumeNavBar != mAttachInfo.mAlwaysConsumeNavBar;
                if (contentInsetsChanged) {
                    mAttachInfo.mContentInsets.set(mPendingContentInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, "Content insets changing to: "
                            + mAttachInfo.mContentInsets);
                }
                if (overscanInsetsChanged) {
                    mAttachInfo.mOverscanInsets.set(mPendingOverscanInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, "Overscan insets changing to: "
                            + mAttachInfo.mOverscanInsets);
                    // Need to relayout with content insets.
                    contentInsetsChanged = true;
                }
                if (stableInsetsChanged) {
                    mAttachInfo.mStableInsets.set(mPendingStableInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, "Decor insets changing to: "
                            + mAttachInfo.mStableInsets);
                    // Need to relayout with content insets.
                    contentInsetsChanged = true;
                }
                if (alwaysConsumeNavBarChanged) {
                    mAttachInfo.mAlwaysConsumeNavBar = mPendingAlwaysConsumeNavBar;
                    contentInsetsChanged = true;
                }
                if (contentInsetsChanged || mLastSystemUiVisibility !=
                        mAttachInfo.mSystemUiVisibility || mApplyInsetsRequested
                        || mLastOverscanRequested != mAttachInfo.mOverscanRequested
                        || outsetsChanged) {
                    mLastSystemUiVisibility = mAttachInfo.mSystemUiVisibility;
                    mLastOverscanRequested = mAttachInfo.mOverscanRequested;
                    mAttachInfo.mOutsets.set(mPendingOutsets);
                    mApplyInsetsRequested = false;
                    dispatchApplyInsets(host);
                }
                if (visibleInsetsChanged) {
                    mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
                    if (DEBUG_LAYOUT) Log.v(mTag, "Visible insets changing to: "
                            + mAttachInfo.mVisibleInsets);
                }
            ...
}
```
这一段主要是 contentInsets 或 visibleInsets 有变动的时候启动过渡动画，先不详细分析了。后面有机会再学习下 Android 窗口中的各种区域。

### 布局窗口后的处理 —— Surface
略

### 最终测量
最终测量与协商测量一样使用 performMeasure 进行，它们的区别在于参数不同。预测量使用配置文件或 Display 的尺寸作为尺寸候选，使用 measureHierarchy() 方法协商确定 MeasureSpec 参数，其测量结果体现了控件树所期望的尺寸。并且在窗口布局阶段将这一尺寸提交给 WMS 希望能将窗口布局为这一尺寸。由于 WMS 需要考虑很多，这一期望不一定能如愿。在最终测量阶段，控件树被迫接受 WMS 布局结果，以最新的窗口尺寸作为 MeasureSpec 参数进行测量。

```java
private void performTraversals() {
                ...
    
                if (!mStopped || mReportNextDraw) {
                boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                        (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
                if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                        || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                        updatedConfiguration) {
                        //使用窗口最新尺寸作为测量参数
                    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

                    if (DEBUG_LAYOUT) Log.v(mTag, "Ooops, something changed!  mWidth="
                            + mWidth + " measuredWidth=" + host.getMeasuredWidth()
                            + " mHeight=" + mHeight
                            + " measuredHeight=" + host.getMeasuredHeight()
                            + " coveredInsetsChanged=" + contentInsetsChanged);

                     //测量
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

                }
            }
            ...
}
```
之所以使用 performMeasure 没有使用 measureHierarchy 是因为 measureHierarchy 包含协商测量，最终测量阶段窗口尺寸已经确定，没有协商余地。

### 总结
* 布局阶段得一进行的原因是控件系统有修改窗口属性的需求：
    * 第一次遍历需要确定窗口尺寸
    * 与测量结果与窗口当前尺寸不一致，需要进行窗口尺寸更新
    * mView 的可见性发生变化
    * LayoutParams 发生变化，需要 WMS 以新的参数进行重新布局
* 最终测量得一进行的原因是：
    * 窗口布局阶段确定的窗口尺寸与控件树所期望的不一致，需要根据新的窗口尺寸进行重新测量。
    
完成这两个阶段之后，performTraversals() 中只剩下布局和绘制了，这两个我们后面再分析。
