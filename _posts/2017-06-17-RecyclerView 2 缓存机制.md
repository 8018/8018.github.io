---
layout:     post
title:      RecyclerView 2 缓存机制
date:       2017-06-17
author:     xflyme
header-img: img/post-bg-2017-06-17.jpg
catalog: true
tags:
    - Android
    - RecyclerView
---


上一节分析了 RecyclerView 的绘制流程，本节我们分析下 RecyclerView 的缓存机制。(本文基于 LinearLayoutManager)

#### 源码分析 
上一节中在 LayoutManager 中我们提到过在 layoutChunk 中的 View view = layoutState.next(recycler);  执行了 itemView 的复用或创建，我们从这里开始看。


```java
View next(RecyclerView.Recycler recycler) {
            if (mScrapList != null) {
                return nextViewFromScrapList();
            }
            final View view = recycler.getViewForPosition(mCurrentPosition);
            mCurrentPosition += mItemDirection;
            return view;
        }
```

在这里首先有一个判断，如果 mScrapList 不为 null 首先从 mScrapList 获取。先看一下 mScrapList：

```java
 /**
         * When LLM needs to layout particular views, it sets this list in which case, LayoutState
         * will only return views from this list and return null if it cannot find an item.
         */
        List<RecyclerView.ViewHolder> mScrapList = null;
```

```java

    private void layoutForPredictiveAnimations(RecyclerView.Recycler recycler,
            RecyclerView.State state, int startOffset,
            int endOffset) {
        ...
        final List<RecyclerView.ViewHolder> scrapList = recycler.getScrapList();
        ...
        mLayoutState.mScrapList = scrapList;
       ...
    }
```
它是在 layoutForPredictiveAnimations 中被赋值的，这里又指向了 Recycler 的 mUnmodifiableAttachedScrap ：

```java
private final List<ViewHolder>
                mUnmodifiableAttachedScrap = Collections.unmodifiableList(mAttachedScrap);
```
有人说 「scrapList 是 RecyclerView 的第五级缓存」，但是这里又指向了 Recycler 中我们接下来要分析的缓存 mAttachedScrap。
**这么做的目的是什么？我们先记住这个问题，以后有机会再分析。**

再回到 next() 方法，接着往下走：

```java

 public View getViewForPosition(int position) {
            return getViewForPosition(position, false);
        }


 View getViewForPosition(int position, boolean dryRun) {
            return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
        }

```

经过两个中间方法之后，最终走到了 tryGetViewHolderForPositionByDeadline 这也是我们今天分析的重点之一。

```java
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                boolean dryRun, long deadlineNs) {
            ...
            ViewHolder holder = null;
            // 0) 如果有变动缓存，从变动缓存里获取
            if (mState.isPreLayout()) {
                holder = getChangedScrapViewForPosition(position);
                fromScrapOrHiddenOrCache = holder != null;
            }
            // 1) 根据 position 从 scrap/hidden list/cache 中获取
            if (holder == null) {
                holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
                ...
            }
            if (holder == null) {
               

                final int type = mAdapter.getItemViewType(offsetPosition);
                // 2) 如果 adapter 复写了 stableIds ，根据 stableId 从 scrap 中获取
                if (mAdapter.hasStableIds()) {
                    holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                            type, dryRun);
                    if (holder != null) {
                        // update position
                        holder.mPosition = offsetPosition;
                        fromScrapOrHiddenOrCache = true;
                    }
                }
                
                if (holder == null && mViewCacheExtension != null) {
                    // 从 mViewCacheExtension 中获取，这个要自己实现
                    final View view = mViewCacheExtension
                            .getViewForPositionAndType(this, position, type);
                    ...
                }
                if (holder == null) { // fallback to pool
                    ...
                    // 从 RecyclerViewPool 中获取
                    holder = getRecycledViewPool().getRecycledView(type);
                    if (holder != null) {
                        holder.resetInternal();
                        if (FORCE_INVALIDATE_DISPLAY_LIST) {
                            invalidateDisplayListInt(holder);
                        }
                    }
                }
                if (holder == null) {
                    ...
                    // 创建 ViewHolder
                    holder = mAdapter.createViewHolder(RecyclerView.this, type);
                    ...
                }
            }

            ...

            boolean bound = false;
            if (mState.isPreLayout() && holder.isBound()) {
                // do not update unless we absolutely have to.
                holder.mPreLayoutPosition = position;
            } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
                if (DEBUG && holder.isRemoved()) {
                    throw new IllegalStateException("Removed holder should be bound and it should"
                            + " come here only in pre-layout. Holder: " + holder
                            + exceptionLabel());
                }
                final int offsetPosition = mAdapterHelper.findPositionOffset(position);
                // 绑定数据
                bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
            }

            ...
            return holder;
        }
```

以上代码做了一些删减，列出了关键步骤并添加了一些注释，下面我们一步步来分析它。

```java
// 0) 如果有变动缓存，从变动缓存里获取
            if (mState.isPreLayout()) {
                holder = getChangedScrapViewForPosition(position);
                fromScrapOrHiddenOrCache = holder != null;
            }
```
**这里跟动画或数据变动有关？** 我们先跳过这段等分析动画或数据更新的时候再来看它。继续往下看：

```java
 // 1) 根据 position 从 scrap/hidden list/cache 中获取
            if (holder == null) {
                holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
                ...
            }
```
这里调用了另一个方法，继续。

```java
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
            final int scrapCount = mAttachedScrap.size();

            // Try first for an exact, non-invalid match from scrap.
            for (int i = 0; i < scrapCount; i++) {
                final ViewHolder holder = mAttachedScrap.get(i);
                if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position
                        && !holder.isInvalid() && (mState.mInPreLayout || !holder.isRemoved())) {
                    holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
                    return holder;
                }
            }

            if (!dryRun) {
                View view = mChildHelper.findHiddenNonRemovedView(position);
                if (view != null) {
                    // 找到隐藏的 view 先 detach 然后复用
                    final ViewHolder vh = getChildViewHolderInt(view);
                    mChildHelper.unhide(view);
                    int layoutIndex = mChildHelper.indexOfChild(view);
                    if (layoutIndex == RecyclerView.NO_POSITION) {
                        throw new IllegalStateException("layout index should not be -1 after "
                                + "unhiding a view:" + vh + exceptionLabel());
                    }
                    mChildHelper.detachViewFromParent(layoutIndex);
                    scrapView(view);
                    vh.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP
                            | ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
                    return vh;
                }
            }

            // Search in our first-level recycled view cache.
            final int cacheSize = mCachedViews.size();
            for (int i = 0; i < cacheSize; i++) {
                final ViewHolder holder = mCachedViews.get(i);
                // invalid view holders may be in cache if adapter has stable ids as they can be
                // retrieved via getScrapOrCachedViewForId
                if (!holder.isInvalid() && holder.getLayoutPosition() == position) {
                    if (!dryRun) {
                        mCachedViews.remove(i);
                    }
                    if (DEBUG) {
                        Log.d(TAG, "getScrapOrHiddenOrCachedHolderForPosition(" + position
                                + ") found match in cache: " + holder);
                    }
                    return holder;
                }
            }
            return null;
        }
```
##### mAttachedScrap
首先第一步是从 mAttachedScrap 中获取，这里要先区分两个概念 detach 和 remove：
> **detach：** 在 ViewGroup 中的实现很简单，只是将 ChildView 从 ParentView 中 ChildView 数组中移除，ChildView 的 parent 设置为 null，可以理解为轻量级的 remove。View 被 detach 一般是临时的，随后可以再 attach。
> **remove：** 真正的移除，不光从 ChildView 数组中移除，和 View 树的各项联系也被切断。

在 RecyclerView 的高度被设置为 wrap_content 的时候，RecyclerView 会经历两次 measure 和 layout，在第二次的时候第一次创建的 ViewHolder 会首先被 detach 然后再 attach 这时 mAttachedScrap 就起了作用。
> 其实高度设置为 match_parent 或 固定高度也会经历两次 measure，只是测量模式为 EXACTLY，onMeasure 中直接返回了。在高度为 wrap_content 的时候**首先用 ItemView 填充以 measure RecyclerView 的高度**，第二次的时候 View 直接就可以复用。

> mAttachedScrap 获取的 ViewHolder 已经被绑定过数据，不需要重新绑定。

**为什么 RecyclerView 会经历两次 measure 和 layout？** 好吧，又挖了一个坑，感觉坑越来越多。

##### mCachedViews

接下来是隐藏的 ItemView 先 detach 然后再复用，这里就不详细解释了。继续往下是 mCachedViews，代码如下：

```java
// Search in our first-level recycled view cache.
            final int cacheSize = mCachedViews.size();
            for (int i = 0; i < cacheSize; i++) {
                final ViewHolder holder = mCachedViews.get(i);
                // invalid view holders may be in cache if adapter has stable ids as they can be
                // retrieved via getScrapOrCachedViewForId
                if (!holder.isInvalid() && holder.getLayoutPosition() == position) {
                    if (!dryRun) {
                        mCachedViews.remove(i);
                    }
                    if (DEBUG) {
                        Log.d(TAG, "getScrapOrHiddenOrCachedHolderForPosition(" + position
                                + ") found match in cache: " + holder);
                    }
                    return holder;
                }
            }
            return null;
```

mCachedViews 是 Recycler 中的一个 ArrayList，它的默认大小是 2。如果存储的数据量大于二，会将最老的 View 移到 RecyclerPool 中。
上面这段代码中，holder 是有效的并且 holder 的 position 和目标 position 一直才会复用，所以它也不需要重新 bind。

经过这两步之后又回到了 tryGetViewHolderForPositionByDeadline，我们接着往下看：
mAdapter.hasStableIds 这个要在 Adapter 中复写相关方法，我们先跳过。

接下来是 mViewCacheExtension ，这个同样要我们自己实现。
再继续往下，分别是 RecyclerViewPool 和 createViewHolder。

##### RecyclerViewPool

我们先看一下 RecyclerViewPool 的关键属性和方法，不是很难就不一一解释了。

```java
public static class RecycledViewPool {
        private static final int DEFAULT_MAX_SCRAP = 5;

        static class ScrapData {
            final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
            int mMaxScrap = DEFAULT_MAX_SCRAP;
            long mCreateRunningAverageNs = 0;
            long mBindRunningAverageNs = 0;
        }
        SparseArray<ScrapData> mScrap = new SparseArray<>();

        private int mAttachCount = 0;

       

        @Nullable
        public ViewHolder getRecycledView(int viewType) {
            final ScrapData scrapData = mScrap.get(viewType);
            if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
                final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
                return scrapHeap.remove(scrapHeap.size() - 1);
            }
            return null;
        }

       
    }
```
不过需要注意的是，从 RecyclerViewPool 中获取的 ViewHolder，数据会被重置，也就是说它需要重新绑定数据。

```java
if (holder != null) {
    holder.resetInternal();
    if (FORCE_INVALIDATE_DISPLAY_LIST) {
        invalidateDisplayListInt(holder);
    }
}
                    
                    
void resetInternal() {
            mFlags = 0;
            mPosition = NO_POSITION;
            mOldPosition = NO_POSITION;
            mItemId = NO_ID;
            mPreLayoutPosition = NO_POSITION;
            mIsRecyclableCount = 0;
            mShadowedHolder = null;
            mShadowingHolder = null;
            clearPayload();
            mWasImportantForAccessibilityBeforeHidden = ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_AUTO;
            mPendingAccessibilityState = PENDING_ACCESSIBILITY_STATE_NOT_SET;
            clearNestedRecyclerViewIfNotNested(this);
        }

```

再往下如果还没有找到可用的 ViewHolder，就只能调用 Adapter 重新创建了，方法我们都很熟悉，就不看了。

上面只是 ViewHolder 的创建和复用流程，那么 ViewHolder 是怎么被回收的呢？我们再看一遍回收流程。


```java
 @Override
    public boolean onTouchEvent(MotionEvent e) {
       ...

        switch (action) {
            ...

            case MotionEvent.ACTION_MOVE: {
               ...

                if (mScrollState == SCROLL_STATE_DRAGGING) {
                    mLastTouchX = x - mScrollOffset[0];
                    mLastTouchY = y - mScrollOffset[1];
                    // 1
                    if (scrollByInternal(
                            canScrollHorizontally ? dx : 0,
                            canScrollVertically ? dy : 0,
                            vtev)) {
                        getParent().requestDisallowInterceptTouchEvent(true);
                    }
                    //2
                    if (mGapWorker != null && (dx != 0 || dy != 0)) {
                        mGapWorker.postFromTraversal(this, dx, dy);
                    }
                }
            } break;

            ...

        return true;
    }
```

View 复用要从滑动说起，也就是上面的 onTouchEvent，关键节点有两个，我们一个个来看。

方法路径如下：
![图一](/img/recyclerview-2-1.png)

我就不一个一个看了，直接从 Recycler.recycleView 开始。

```java
public void recycleView(@NonNull View view) {
            // This public recycle method tries to make view recycle-able since layout manager
            // intended to recycle this view (e.g. even if it is in scrap or change cache)
            ViewHolder holder = getChildViewHolderInt(view);
            if (holder.isTmpDetached()) {
                removeDetachedView(view, false);
            }
            if (holder.isScrap()) {
                holder.unScrap();
            } else if (holder.wasReturnedFromScrap()) {
                holder.clearReturnedFromScrapFlag();
            }
            recycleViewHolderInternal(holder);
        }
```
recycleViewHolderInternal
```java
void recycleViewHolderInternal(ViewHolder holder) {
            ...
            boolean cached = false;
            boolean recycled = false;
            ...
            if (forceRecycle || holder.isRecyclable()) {
                if (mViewCacheMax > 0
                        && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                        | ViewHolder.FLAG_REMOVED
                        | ViewHolder.FLAG_UPDATE
                        | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
                    // Retire oldest cached view
                    // 如果 mCachedViews 已经满了，将最老的移到 RecyclerViewPool
                    int cachedViewSize = mCachedViews.size();
                    if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                        recycleCachedViewAt(0);
                        cachedViewSize--;
                    }

                    int targetCacheIndex = cachedViewSize;
                    if (ALLOW_THREAD_GAP_WORK
                            && cachedViewSize > 0
                            && !mPrefetchRegistry.lastPrefetchIncludedPosition(holder.mPosition)) {
                        // when adding the view, skip past most recently prefetched views
                        int cacheIndex = cachedViewSize - 1;
                        while (cacheIndex >= 0) {
                            int cachedPos = mCachedViews.get(cacheIndex).mPosition;
                            if (!mPrefetchRegistry.lastPrefetchIncludedPosition(cachedPos)) {
                                break;
                            }
                            cacheIndex--;
                        }
                        targetCacheIndex = cacheIndex + 1;
                    }
                    // 将目标添加到 mCachedViews
                    mCachedViews.add(targetCacheIndex, holder);
                    cached = true;
                }
                if (!cached) {
                    // 如果上面没有成功 添加到 RecyclerViewPool 中
                    addViewHolderToRecycledViewPool(holder, true);
                    recycled = true;
                }
            } else {
              ...
        }

```

这里我们可以看到：
> mCachedViews 优先级比 RecyclerViewPool 高，如果 mCachedViews 已满，将最老的那个放到 RecyclerViewPool 中，**可以理解为一个先进先出的队列**。

让我们再回到 onTouchEvent，看另一个可能触发 ViewHolder 生成与缓存的方法：

```java
if (mGapWorker != null && (dx != 0 || dy != 0)) {
    mGapWorker.postFromTraversal(this, dx, dy);
}
                    
```

```java
 void postFromTraversal(RecyclerView recyclerView, int prefetchDx, int prefetchDy) {
        if (recyclerView.isAttachedToWindow()) {
            if (RecyclerView.DEBUG && !mRecyclerViews.contains(recyclerView)) {
                throw new IllegalStateException("attempting to post unregistered view!");
            }
            if (mPostTimeNs == 0) {
                mPostTimeNs = recyclerView.getNanoTime();
                recyclerView.post(this);
            }
        }

        recyclerView.mPrefetchRegistry.setPrefetchVector(prefetchDx, prefetchDy);
    }
```
recyclerView.post(this); 这个方法我们都很熟悉，它是什么意思呢？其实就是向 Handler 发送一个消息，然后等待执行。

但是在 RecyclerView 向上滑动的时候必定会触发 ItemView 的 invalidate，这个时候会向 Handler 消息队列中插入一个同步屏障：

```java
void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```

所以 RecyclerView.post 向主线程发送的消息会在停止滑动之后再执行。

![图二](/img/recyclerview-2-2.png)

![图三](/img/recyclerview-2-3.png)

上图中，图一中 item 17 已经有部分显示，这时候滑动停止。然后之前发送的回掉触发了 item 18 的生成与回收。新生成的 item 被放到 mCachedViews 里。

也就是说 mCachedViews 存储的不只是被回收的 ViewHolder 还有可能是即将显示而被提前创建出来的 item。

以上，RecyclerView 的缓存机制已经分析完了。

#### 总结

直接放网上的一个图吧

![图四](/img/recyclerview-2-4.jpg)

> 不过有一点需要注意，mCacheView 的 size() 并不是一直都是 2。GapWorker 有可能将它更新为 3.
