---
layout:     post
title:      Android 事件系列-2 ViewGroup 中的事件分发
date:       2017-03-26
author:     xflyme
header-img: img/post-bg-2017-03-26.jpeg
catalog: true
tags:
    - Android
    - 事件分发
    - 源码分析
---


上一篇中我们分析了 [View 中的事件分发](https://app.yinxiang.com/shard/s12/nl/2184113/4818638f-50aa-4e54-aa02-2a4e75f38262/),Android 中直接继承 View 的控件是最小单位，不能包含其他子 View，其中的分发逻辑也比较简单。本节研究下 ViewGroup 中的事件分发。

### ViewGroup 事件传递源码分析

#### Demo

ViewGroup 中的 dispatchTouchEvent 逻辑较多，我们先看一个 Demo，从这个 Demo 深入分析 ViewGroup#dispatchTouchEvent。

定义个 LinearLayout 复写相关方法：
```java
public class MyLinearLayout extends LinearLayout {

    final String TAG = "MyLinearLayout";

    public MyLinearLayout(Context context) {
        super(context);
    }

    public MyLinearLayout(Context context,  AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        Log.e(TAG,"dispatchTouchEvent="+ev.getAction());
        return super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        Log.e(TAG,"onInterceptTouchEvent="+ev.getAction());
        return super.onInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.e(TAG,"onTouchEvent="+event.getAction());
        return super.onTouchEvent(event);
    }
}
```
定义一个 Button 复写相关方法：

```java
public class MyButton extends android.support.v7.widget.AppCompatButton {
    static final String TAG = "MyButton";
    public MyButton(Context context) {
        super(context);
    }

    public MyButton(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent event) {
        Log.e(TAG,"dispatchTouchEvent="+event.getAction());
        return super.dispatchTouchEvent(event);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.e(TAG,"onTouchEvent="+event.getAction());
        return super.onTouchEvent(event);
    }
}
```
布局文件
```xml
<?xml version="1.0" encoding="utf-8"?>
<com.mrball.eventdemo.MyLinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:id="@+id/linearlayout"
    android:gravity="center"
    tools:context=".MainActivity">

    <com.mrball.eventdemo.MyButton
        android:layout_width="match_parent"
        android:text="button"
        android:id="@+id/button"
        android:clickable="false"
        android:layout_height="wrap_content" />

</com.mrball.eventdemo.MyLinearLayout>
```
MainActivity：
```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener, View.OnTouchListener {

    private static final String TAG = "EventDemo";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        findViewById(R.id.button).setOnTouchListener(this);
        findViewById(R.id.button).setOnClickListener(this);
        findViewById(R.id.linearlayout).setOnTouchListener(this);
        findViewById(R.id.linearlayout).setOnClickListener(this);

    }

    @Override
    public void onClick(View v) {
        Log.e(TAG,"onClick=="+v.toString());
    }

    @Override
    public boolean onTouch(View v, MotionEvent event) {
        Log.e(TAG,"onTouch==="+event.getAction()+"===v==="+v.toString());
        return false;
    }
}
```
点击按钮，点击的时候滑动一下然后得到如下日志：
```xml
2019-06-09 15:24:41.462 8954-8954/com.mrball.eventdemo E/MyLinearLayout: dispatchTouchEvent=0
2019-06-09 15:24:41.462 8954-8954/com.mrball.eventdemo E/MyLinearLayout: onInterceptTouchEvent=0
2019-06-09 15:24:41.462 8954-8954/com.mrball.eventdemo E/MyButton: dispatchTouchEvent=0
2019-06-09 15:24:41.462 8954-8954/com.mrball.eventdemo E/EventDemo: onTouch===0===v===com.mrball.eventdemo.MyButton{6f4a64f VFED..C.. ........ 0,1132-1440,1300 #7f070022 app:id/button}
2019-06-09 15:24:41.463 8954-8954/com.mrball.eventdemo E/MyButton: onTouchEvent=0
2019-06-09 15:24:41.548 8954-8954/com.mrball.eventdemo E/MyLinearLayout: dispatchTouchEvent=2
2019-06-09 15:24:41.548 8954-8954/com.mrball.eventdemo E/MyLinearLayout: onInterceptTouchEvent=2
2019-06-09 15:24:41.548 8954-8954/com.mrball.eventdemo E/MyButton: dispatchTouchEvent=2
2019-06-09 15:24:41.548 8954-8954/com.mrball.eventdemo E/EventDemo: onTouch===2===v===com.mrball.eventdemo.MyButton{6f4a64f VFED..C.. ...P.... 0,1132-1440,1300 #7f070022 app:id/button}
2019-06-09 15:24:41.548 8954-8954/com.mrball.eventdemo E/MyButton: onTouchEvent=2
2019-06-09 15:24:41.565 8954-8954/com.mrball.eventdemo E/MyLinearLayout: dispatchTouchEvent=2
2019-06-09 15:24:41.565 8954-8954/com.mrball.eventdemo E/MyLinearLayout: onInterceptTouchEvent=2
2019-06-09 15:24:41.565 8954-8954/com.mrball.eventdemo E/MyButton: dispatchTouchEvent=2
2019-06-09 15:24:41.565 8954-8954/com.mrball.eventdemo E/EventDemo: onTouch===2===v===com.mrball.eventdemo.MyButton{6f4a64f VFED..C.. ...P.... 0,1132-1440,1300 #7f070022 app:id/button}
2019-06-09 15:24:41.565 8954-8954/com.mrball.eventdemo E/MyButton: onTouchEvent=2
2019-06-09 15:24:41.574 8954-8954/com.mrball.eventdemo E/MyLinearLayout: dispatchTouchEvent=1
2019-06-09 15:24:41.574 8954-8954/com.mrball.eventdemo E/MyLinearLayout: onInterceptTouchEvent=1
2019-06-09 15:24:41.575 8954-8954/com.mrball.eventdemo E/MyButton: dispatchTouchEvent=1
2019-06-09 15:24:41.575 8954-8954/com.mrball.eventdemo E/EventDemo: onTouch===1===v===com.mrball.eventdemo.MyButton{6f4a64f VFED..C.. ...P.... 0,1132-1440,1300 #7f070022 app:id/button}
2019-06-09 15:24:41.575 8954-8954/com.mrball.eventdemo E/MyButton: onTouchEvent=1
2019-06-09 15:24:41.578 8954-8954/com.mrball.eventdemo E/EventDemo: onClick==com.mrball.eventdemo.MyButton{6f4a64f VFED..C.. ...P.... 0,1132-1440,1300 #7f070022 app:id/button}

```

点击 Button 以外的区域并小幅滑动一下：
```xml
2019-06-09 15:27:31.025 8954-8954/com.mrball.eventdemo E/MyLinearLayout: dispatchTouchEvent=0
2019-06-09 15:27:31.025 8954-8954/com.mrball.eventdemo E/MyLinearLayout: onInterceptTouchEvent=0
2019-06-09 15:27:31.025 8954-8954/com.mrball.eventdemo E/EventDemo: onTouch===0===v===com.mrball.eventdemo.MyLinearLayout{37db861 VFE...C.. ........ 0,0-1440,2432 #7f07004d app:id/linearlayout}
2019-06-09 15:27:31.025 8954-8954/com.mrball.eventdemo E/MyLinearLayout: onTouchEvent=0
2019-06-09 15:27:31.033 8954-8954/com.mrball.eventdemo E/MyLinearLayout: dispatchTouchEvent=2
2019-06-09 15:27:31.033 8954-8954/com.mrball.eventdemo E/EventDemo: onTouch===2===v===com.mrball.eventdemo.MyLinearLayout{37db861 VFE...C.. ...P.... 0,0-1440,2432 #7f07004d app:id/linearlayout}
2019-06-09 15:27:31.033 8954-8954/com.mrball.eventdemo E/MyLinearLayout: onTouchEvent=2
2019-06-09 15:27:31.061 8954-8954/com.mrball.eventdemo E/MyLinearLayout: dispatchTouchEvent=2
2019-06-09 15:27:31.061 8954-8954/com.mrball.eventdemo E/EventDemo: onTouch===2===v===com.mrball.eventdemo.MyLinearLayout{37db861 VFE...C.. ...P.... 0,0-1440,2432 #7f07004d app:id/linearlayout}
2019-06-09 15:27:31.061 8954-8954/com.mrball.eventdemo E/MyLinearLayout: onTouchEvent=2
2019-06-09 15:27:31.078 8954-8954/com.mrball.eventdemo E/MyLinearLayout: dispatchTouchEvent=2
2019-06-09 15:27:31.078 8954-8954/com.mrball.eventdemo E/EventDemo: onTouch===2===v===com.mrball.eventdemo.MyLinearLayout{37db861 VFE...C.. ...P.... 0,0-1440,2432 #7f07004d app:id/linearlayout}
2019-06-09 15:27:31.078 8954-8954/com.mrball.eventdemo E/MyLinearLayout: onTouchEvent=2
2019-06-09 15:27:31.094 8954-8954/com.mrball.eventdemo E/MyLinearLayout: dispatchTouchEvent=2
2019-06-09 15:27:31.094 8954-8954/com.mrball.eventdemo E/EventDemo: onTouch===2===v===com.mrball.eventdemo.MyLinearLayout{37db861 VFE...C.. ...P.... 0,0-1440,2432 #7f07004d app:id/linearlayout}
2019-06-09 15:27:31.094 8954-8954/com.mrball.eventdemo E/MyLinearLayout: onTouchEvent=2
2019-06-09 15:27:31.840 8954-8954/com.mrball.eventdemo E/MyLinearLayout: dispatchTouchEvent=1
2019-06-09 15:27:31.840 8954-8954/com.mrball.eventdemo E/EventDemo: onTouch===1===v===com.mrball.eventdemo.MyLinearLayout{37db861 VFE...C.. ...P.... 0,0-1440,2432 #7f07004d app:id/linearlayout}
2019-06-09 15:27:31.840 8954-8954/com.mrball.eventdemo E/MyLinearLayout: onTouchEvent=1
2019-06-09 15:27:31.840 8954-8954/com.mrball.eventdemo E/EventDemo: onClick==com.mrball.eventdemo.MyLinearLayout{37db861 VFE...C.. ...P.... 0,0-1440,2432 #7f07004d app:id/linearlayout}

```
你会发现除了 ACTION_DOWN 以外，MOVE 和 UP 事件没有触发 OnInterceptTouchEvent。
下面我们看下源码，从源码中分析整个分发流程。

#### 1、先从 dispatchTouchEvent 说起

ViewGroup#dispatchTouchEvent

```java

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
        }

        // If the event targets the accessibility focused view and this is it, start
        // normal event dispatch. Maybe a descendant is what will handle the click.
        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }

        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

            // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }

            // Check for cancelation.
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // Update list of touch targets for pointer down, if needed.
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            //定义一个变量 记录是否已经完成派发，由于确定派发目标使用了子控件的 dispatchTouchEvent，因此确定派发目标的时候实际上已经完成派发，这种情况下该变量将被设置为 true
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {

                // If the event is targeting accessibility focus we give it to the
                // view that has accessibility focus and if it does not handle it
                // we clear the flag and dispatch the event to all children as usual.
                // We are looking up the accessibility focused host to avoid keeping
                // state since these events are very rare.
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    //通过索引号获得触控点 id，当找到派发目标后，会将这个 idBitsToAssign 和目标 View 一起生成一个 TouchTarget
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // Clean up earlier touch targets for this pointer id in case they
                    // have become out of sync.
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                    //触摸事件是基于位置进行派发目标的查找，要先获取事件的坐标
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        // Find a child that can receive the event.
                        // Scan children from front to back.
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);

                            // If there is a view that has accessibility focus we want it
                            // to get the event first and if not handled we will perform a
                            // normal dispatch. We may do a double iteration but this is
                            // safer given the timeframe.
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }
//检查点击事件是否在控件的坐标范围内，如果不在跳过 查找下一个控件
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

//查找当前控件是否已经在 mFirstTouchTarget 链表中，如果在表明该控件已经在接收一组事件，ViewGroup 会默认该控件对本组事件也感兴趣，将本组事件的目标设置为当前控件，并终止派发目标的查找。
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                // Child is already receiving touch within its bounds.
                                // Give it the new pointer in addition to the ones it is handling.
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            // 使用 dispatchTransformedTouchEvent 尝试将事件派发给当前子控件，
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                //该子控件决定接受本组事件，将子控件添加到目标链表中。
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                            // The accessibility focus didn't handle the event, so clear
                            // the flag and do a normal dispatch to all children.
                            ev.setTargetAccessibilityFocus(false);
                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

            // Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                //没有找到子控件接受事件，自己处理事件
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    //如果 touchTarget 是新确定的 touchTarget，在确定目标的过程中已经将事件分发出去，不需要再派发。
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                                
                                //将事件分发给子控件
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }

            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }
```

这段代码更长，我们一段一段找重点来看：
```java

            // Handle an initial down.
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                // Throw away all previous state when starting a new touch gesture.
                // The framework may have dropped the up or cancel event for the previous gesture
                // due to an app switch, ANR, or some other state change.
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }
```
这段代码中调用了 cancelAndClearTouchTargets()

#cancelAndClearTouchTargets
```java

    /**
     * Cancels and clears all touch targets.
     */
    private void cancelAndClearTouchTargets(MotionEvent event) {
        if (mFirstTouchTarget != null) {
            boolean syntheticEvent = false;
            if (event == null) {
                final long now = SystemClock.uptimeMillis();
                event = MotionEvent.obtain(now, now,
                        MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                event.setSource(InputDevice.SOURCE_TOUCHSCREEN);
                syntheticEvent = true;
            }

            for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
                resetCancelNextUpFlag(target.child);
                dispatchTransformedTouchEvent(event, true, target.child, target.pointerIdBits);
            }
            clearTouchTargets();

            if (syntheticEvent) {
                event.recycle();
            }
        }
    }
```
这段代码中会判断 mFirstTarget，它是一个链表，如果该属性不为 null，它会做两件事
1. 遍历 mFirstTarget 链表，对链表的每一个节点，执行 dispatchTransformedTouchEvent 方法。
2. clearTouchTargets。 
先看第一个：
ViewGroup#dispatchTransformedTouchEvent
```java
 /**
     * Transforms a motion event into the coordinate space of a particular child view,
     * filters out irrelevant pointer ids, and overrides its action if necessary.
     * If child is null, assumes the MotionEvent will be sent to this ViewGroup instead.
     */
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        // Canceling motions is a special case.  We don't need to perform any transformations
        // or filtering.  The important part is the action, not the contents.
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        // Calculate the number of pointers to deliver.
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        // If for some reason we ended up in an inconsistent state where it looks like we
        // might produce a motion event with no pointers in it, then drop the event.
        if (newPointerIdBits == 0) {
            return false;
        }

        // If the number of pointers is the same and we don't need to perform any fancy
        // irreversible transformations, then we can reuse the motion event for this
        // dispatch as long as we are careful to revert any changes we make.
        // Otherwise we need to make a copy.
        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);

                    handled = child.dispatchTouchEvent(event);

                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

        // Perform any necessary transformations and dispatch.
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;
    }

```
这个方法我们下面还要分析，现在我们先看 cancel=true 的情况下会执行那些逻辑：
```java
final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }
```
昨天我们分析了，View 中的事件分发 [Android 事件系列-1 View 中的事件分发](https://app.yinxiang.com/shard/s12/nl/2184113/4818638f-50aa-4e54-aa02-2a4e75f38262/) 其中 dispatchTouchEvent 中有一段代码：
```java
 // Clean up after nested scrolls if this is the end of a gesture;
        // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
        // of the gesture.
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }
```
ACTION_UP 或 ACTION_CANCEL 的情况下，停止嵌套滑动。
接着看第二件事：
ViewGroup#
```java
 /**
     * Clears all touch targets.
     */
    private void clearTouchTargets() {
        TouchTarget target = mFirstTouchTarget;
        if (target != null) {
            do {
                TouchTarget next = target.next;
                target.recycle();
                target = next;
            } while (target != null);
            mFirstTouchTarget = null;
        }
    }
```
逻辑也很简单，遍历所有节点，执行 recycle() 方法。
#### 小结
至此，我们分析了 ViewGroup#dispatchTouchEvent 中的第一个主要逻辑：如果当前事件是 ACTION_DOWN，清空 touchTargets、重置 touchState。

现在回到 ViewGroup#dispatchTouchEvent 接着往下走
```java
  // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }

```
这段代码使用 intercepted 变量检查 ViewGroup 是否拦截事件。直接看图：

![图1](/img/event-2-1.png)

1. 如果没有 touch targets 而且 事件也不是 ACTION_DOWN ，这个 view group 可以继续拦截事件
2. touch targets 不为 null 或 事件是 ACTION_DOWN 继续判断 disallowIntercept 标识位。
3. disallowIntercept 为真，即不允许拦截，直接将 intercepted 设置为 false。
4. disallowIntercept 为假，即允许拦截，接收 onInterceptTouchEvent 的返回值。


看一下，onInterceptTouchEvent：

```java
   public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
                && ev.getAction() == MotionEvent.ACTION_DOWN
                && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
                && isOnScrollbarThumb(ev.getX(), ev.getY())) {
            return true;
        }
        return false;
    }
```
这段代码我们可以直接理解为默认返回false，即不拦截。

接着回到 dispatchTouchEvent 往下看：
```java
  // If intercepted, start normal event dispatch. Also if there is already
            // a view that is handling the gesture, do normal event dispatch.
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }
```
这两行注释很详细，如果被拦截了，开始一个正常事件的分发。如果已经有一个 view 在处理这一系列事件，也做正常事件分发。
继续往下看：
```java
final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
```
从这行往下，都是事件分发，也就是找到处理该事件的目标 View 将事件交给它处理。

接着 if (!canceled && !intercepted) 判断表明，事件不是ACTION_CANCEL并且ViewGroup的拦截标志位intercepted为false(不拦截)则会进入其中。

接着往下：

```java
newTouchTarget = getTouchTarget(child);

 private TouchTarget getTouchTarget(@NonNull View child) {
        for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
            if (target.child == child) {
                return target;
            }
        }
        return null;
    }
```
通过getTouchTarget去查找当前子View是否在mFirstTouchTarget.next这条target链中的某一个targe中，如果在则返回这个target，否则返回null。

如果这个 target 存在，则 break 跳出这个循环，ACTION_DOWN 的时候 mFirstTouchTarget 还未被赋值，自然会返回 null 。
接着往下：

```java
if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // Child wants to receive touch within its bounds.
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    // childIndex points into presorted list, find original index
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
```
这是比较关键的一段代码，dispatchTransformedTouchEvent 将事件传递给子 View，调用 child 的 dispatchTouchEvent 方法。dispatchTouchEvent 和 dispatchTransformedTouchEvent 配合基本上就是一个递归，直到 child 是一个 View 的时候才能跳出循环。

> 因为在dispatchTransformedTouchEvent()会调用递归调用dispatchTouchEvent()和onTouchEvent()，所以dispatchTransformedTouchEvent()的返回值实际上是由onTouchEvent()决定的。简单地说onTouchEvent()是否消费了Touch事件的返回值决定了dispatchTransformedTouchEvent()的返回值，从而决定mFirstTouchTarget是否为null，进一步决定了ViewGroup是否处理Touch事件

```java
newTouchTarget = addTouchTarget(child, idBitsToAssign);
```
这一行为 mFirstTouchTarget 赋值。

#### ACTION_DOWN 之外的其他事件
```java
// Dispatch to touch targets.
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }
```
根据前面的分析，如果子 View 接收事件，mFirstTouchTarget 被赋值，如果子 View 不接收事件 mFirstTouchTarget 为 null。
其中第一个逻辑，  mFirstTouchTarget 为 null 走了  handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
这里主要看第三个参数，也就是 child ，当 child 为 null 的时候
handled = super.dispatchTouchEvent(transformedEvent);
这是就是上篇文章中分析的 View 中的 dispatchTouchEvent 方法，不再一一解释了。

mFirstTouchTarget不为null时，也就是说找到了可以消费Touch事件的子View且后续Touch事件可以传递到该子View。可以看见在源码的else中对于非ACTION_DOWN事件继续传递给目标子组件进行处理，依然是递归调用 dispatchTransformedTouchEvent()方法来实现的处理。

到此ViewGroup的dispatchTouchEvent方法分析完毕。

### 总结：
1. 如果 ViewGroup 方法中拦截了，即 OnInterceptTouchEvent 返回 true，intercepted 标志 为 true。这个时候 mFirstTouchTarget 没有被赋值，走 dispatchTransformedTouchEvent 自己处理。
2. 没有拦截的话 dispatchTouchEvent 配合 dispatchTransformedTouchEvent 递归找到目标 View。

回到我们开始的 Demo，点击 Button 的时候：
1. 首先分发的是 ACTION_DOWN 事件，走进 onInterceptTouchEvent。
2. 后续 MOVE 和 UP 事件，mFirstTouchTarget 已经被赋值，所以依然走进 onInterceptTouchEvent。

点击 Button 以外的区域的时候：
1. 首先分发的是 ACTION_DOWN 事件，走进 onInterceptTouchEvent。
2. 后续事件不是 ACTION_DOWN，mFirstTouchTarget 也没被赋值，所以不走 onInterceptTouchEvent。
