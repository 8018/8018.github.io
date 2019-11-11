---
layout:     post
title:      Android 控件-7 ViewRootImpl - 5 performTraversals 之绘制&总结
date:       2015-07-12
author:     xflyme
header-img: img/post-bg-2015-07-12.jpg
catalog: true
tags:
    - Android
    - 控件
    - 源码分析
---


经过前面几个阶段，每个控件都已经确定好了自己的尺寸与位置，接下来就是最终的绘制阶段。

```java
public void performTraversals(){
    ...
     mFirst = false;
        mWillDrawSoon = false;
        mNewSurfaceNeeded = false;
        mActivityRelaunched = false;
        mViewVisibility = viewVisibility;
        mHadWindowFocus = hasWindowFocus;

        if (hasWindowFocus && !isInLocalFocusMode()) {
            final boolean imTarget = WindowManager.LayoutParams
                    .mayUseInputMethod(mWindowAttributes.flags);
            if (imTarget != mLastWasImTarget) {
                mLastWasImTarget = imTarget;
                InputMethodManager imm = InputMethodManager.peekInstance();
                if (imm != null && imTarget) {
                    imm.onPreWindowFocus(mView, hasWindowFocus);
                    imm.onPostWindowFocus(mView, mView.findFocus(),
                            mWindowAttributes.softInputMode,
                            !mHasHadWindowFocus, mWindowAttributes.flags);
                }
            }
        }

        // Remember if we must report the next draw.
        if ((relayoutResult & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
            reportNextDraw();
        }

        boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;

        if (!cancelDraw && !newSurface) {
            if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).startChangingAnimations();
                }
                mPendingTransitions.clear();
            }

            performDraw();
        } else {
            if (isViewVisible) {
                // Try again
                scheduleTraversals();
            } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).endChangingAnimations();
                }
                mPendingTransitions.clear();
            }
        }

        mIsInTraversal = false;
    ...
}
```
由以上代码可知，绘制阶段最终调用了 performDraw() 进入控件树绘制。View 的绘制这里就不深入了。

### 总结
至此，ViewRootImpl.performTraversals() 已经分析完毕，其整个过程可以参考下图：

![图1](/img/android-view-7-1.jpeg)

可见，前四个阶段以 layoutRequested 为执行条件，即 requestLayout() 被调用过。这是因为这几个阶段的目的是确定控件的位置与尺寸。当一个或多个控件有改变位置和尺寸的需求时，才有执行的必要。

即使 layoutRequest() 未被调用过，绘制阶段也会被执行。因为很多时候需要在不改变控件尺寸和位置的情况下重绘。
