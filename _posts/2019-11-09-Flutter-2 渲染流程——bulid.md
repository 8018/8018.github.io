---
layout:     post
title:      Flutter-2 渲染流程——bulid
date:       2019-11-09
author:     xflyme
header-img: img/post-bg-2019-11-09.jpeg
catalog: true
tags:
    - Flutter
    - 源码分析
---


### 前言
前一节[《Widget Element 和 RenderObject》](https://xfly.me/2019/11/01/Flutter-1-Widget-Element-%E5%92%8C-RenderObject/) 我们分析了 Widget Element 和 RenderObject 之间的关系。后续继续分析 Flutter 的渲染流程。

![图一](/img/flutter-2-1.png)

从这张图可以看出，整个渲染流程分为多个阶段，今天我们来看第一个阶段——build。

### 请求 Vsync
当我们调用 State.setState() 会引起 StateFullWidget 的 rebuild，我们从 setState（）开始。

State#setState();
```dart
 @protected
  void setState(VoidCallback fn) {
   ...
    _element.markNeedsBuild();
  }
```

这里会调用 Element 的 markNeedsBuild()。

Element#markNeedsBuild()
```dart
void markNeedsBuild() {
   
   // 非 active 状态直接返回
    if (!_active)
      return;
   //如果已经标记为 dirty 也直接返回
    if (dirty)
      return;
    _dirty = true;
    owner.scheduleBuildFor(this);
  }
```
标记为 dirty 之后会调用 BuildOwner 的 scheduleBuildFor() 方法。按字面意思理解应该是请求 Vsync 信号，我们来看一下它：

BuildOwner#scheduleBuildFor(Element element)
```dart
void scheduleBuildFor(Element element) {

    if (element._inDirtyList) {
      _dirtyElementsNeedsResorting = true;
      return;
    }
    if (!_scheduledFlushDirtyElements && onBuildScheduled != null) {
      _scheduledFlushDirtyElements = true;
      onBuildScheduled();
    }
    _dirtyElements.add(element);
    element._inDirtyList = true;
    
  }
```
这里做了两个操作：onBuildScheduled() 和把 Element 添加进 BuildOwner 的 _dirtyElements 数组。

而 onBuildScheduled() 是一个回调，它是在 Binding 时被设置的：
```dart
mixin WidgetsBinding on BindingBase, SchedulerBinding, GestureBinding, RendererBinding, SemanticsBinding {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    buildOwner.onBuildScheduled = _handleBuildScheduled;
    ...
  }
  
```

继续看一下这个回调函数里做了什么：
```dart
void _handleBuildScheduled() {
    // If we're in the process of building dirty elements, then changes
    // should not trigger a new frame.
    ensureVisualUpdate();
  }
```
这里有一段注释：
> 如果已经在 building 流程中，没有必要触发新的一帧。
继续：

SchedulerBinding.ensureVisualUpdate()
```dart
void ensureVisualUpdate() {
    switch (schedulerPhase) {
      case SchedulerPhase.idle:
      case SchedulerPhase.postFrameCallbacks:
        scheduleFrame();
        return;
      case SchedulerPhase.transientCallbacks:
      case SchedulerPhase.midFrameMicrotasks:
      case SchedulerPhase.persistentCallbacks:
        return;
    }
  }
```
这个方法会判断当前的状态，如果是 idle 或 postFrameCallbacks 则请求新的一帧，如果是其他状态说明正在渲染流程中直接返回。

SchedulerBinding.scheduleFrame()
```dart
 void scheduleFrame() {
    if (_hasScheduledFrame || !_framesEnabled)
      return;
    window.scheduleFrame();
    _hasScheduledFrame = true;
  }
```

这里首先首先做一个判断 _hasScheduledFrame 是否已经请求了。_framesEnabled 代表 App 当前的状态，也就是说当前 App 是否允许刷新。
具体含义不细说了（我也没弄明白），以后有机会补上。

接下来就是调用 window.scheduleFrame() 以及设置标志位。

Window.scheduleFrame()
```dart
/// Requests that, at the next appropriate opportunity, the [onBeginFrame] and [onDrawFrame] callbacks be invoked.
  void scheduleFrame() native 'Window_scheduleFrame';
```
这是一个 native 方法，注释中有说明垂直同步事件来了之后 onBeginFrame 和 onDrawFrame 会被触发。

### 收到 Vsync 之后

在 SchedulerBinding 中这两个方法分别是 _handleBeginFrame 和 _handleDrawFrame。

SchedulerBinding.handleBeginFrame()
```dart
void handleBeginFrame(Duration rawTimeStamp) {
    Timeline.startSync('Frame', arguments: timelineWhitelistArguments);
    _firstRawTimeStampInEpoch ??= rawTimeStamp;
    _currentFrameTimeStamp = _adjustForEpoch(rawTimeStamp ?? _lastRawTimeStamp);
    if (rawTimeStamp != null)
      _lastRawTimeStamp = rawTimeStamp;

    _hasScheduledFrame = false;
    try {
      // TRANSIENT FRAME CALLBACKS
      Timeline.startSync('Animate', arguments: timelineWhitelistArguments);
      _schedulerPhase = SchedulerPhase.transientCallbacks;
      final Map<int, _FrameCallbackEntry> callbacks = _transientCallbacks;
      _transientCallbacks = <int, _FrameCallbackEntry>{};
      callbacks.forEach((int id, _FrameCallbackEntry callbackEntry) {
        if (!_removedIds.contains(id))
          _invokeFrameCallback(callbackEntry.callback, _currentFrameTimeStamp, callbackEntry.debugStack);
      });
      _removedIds.clear();
    } finally {
      _schedulerPhase = SchedulerPhase.midFrameMicrotasks;
    }
  }
```
注释里写的很清楚，这里主要处理一些 TRANSIENT 「临时的」回调，代码中 _transientCallbacks 被置为空了。如果下一帧仍需要回调，需要再次设置。
这些回调跟动画有关，后续我们再分析。

这个方法完成之后会走 handleDrawFrame()。

SchedulerBinding.handleDrawFrame()
```dart
void handleDrawFrame() {
    try {
      // PERSISTENT FRAME CALLBACKS
      _schedulerPhase = SchedulerPhase.persistentCallbacks;
      for (FrameCallback callback in _persistentCallbacks)
        _invokeFrameCallback(callback, _currentFrameTimeStamp);

      // POST-FRAME CALLBACKS
      _schedulerPhase = SchedulerPhase.postFrameCallbacks;
      final List<FrameCallback> localPostFrameCallbacks =
          List<FrameCallback>.from(_postFrameCallbacks);
      _postFrameCallbacks.clear();
      for (FrameCallback callback in localPostFrameCallbacks)
        _invokeFrameCallback(callback, _currentFrameTimeStamp);
    } finally {
      _schedulerPhase = SchedulerPhase.idle;
      Timeline.finishSync(); // end the Frame

      _currentFrameTimeStamp = null;
    }
  }
```

handleDrawFrame 中处理了两类回调 Persistent 和 Post-Frame 。“Persistent”字面意思是永久的。这类回调一旦注册以后是不能取消的。主要用来驱动渲染流水线。渲染流水线的构建（build），布局（layout）和绘制（paint）阶段都是在其中一个回调里的。

“Post-Frame”回调主要是在新帧渲染完成以后的一类调用，此类回调只会被调用一次。

在运行“Persistent”回调之前_schedulerPhase状态变为SchedulerPhase.persistentCallbacks。在运行“Post-Frame”回调之前_schedulerPhase状态变为SchedulerPhase.postFrameCallbacks。最终状态变为SchedulerPhase.idle。

这里我们关注的一个回调是   _handlePersistentFrameCallback(Duration timeStamp)，它是在 RenderBinding 绑定时注册的。

RenderBinding._handlePersistentFrameCallback()
```dart
 void _handlePersistentFrameCallback(Duration timeStamp) {
    drawFrame();
  }
```
可以看到最终会调用 drawFrame()，WidgetsBinding mixin 了 RenderBinding。这两个类里的 drawFrame 都会被调用，但是它们侧重点不同，WidgetsBinding 作用于 build 阶段，RenderBinding 作用于 layout 以及渲染阶段。

WidgetsBinding.drawFrame()
```dart
 @override
  void drawFrame() {
     try {
      if (renderViewElement != null)
        buildOwner.buildScope(renderViewElement);
      super.drawFrame();
      buildOwner.finalizeTree();
    } finally {
        return true;
      }());
    }
  }
```
这里调用了 BuildOwner 的 buildScope() 参数 renderViewElement 其实是 Element 树的根节点。

随后还调用了父类的方法，也就是 layout 以及 paint 等，我们后续再分析。

最后是 buildOwner.finalizeTree() ，这里做了什么，先挖个坑，后续再分析。

### build 阶段
```dart
void buildScope(Element context, [ VoidCallback callback ]) {
    if (callback == null && _dirtyElements.isEmpty)
      return;
      
    try {
      _scheduledFlushDirtyElements = true;
      //排序
      _dirtyElements.sort(Element._sort);
      _dirtyElementsNeedsResorting = false;
      int dirtyCount = _dirtyElements.length;
      int index = 0;
      while (index < dirtyCount) {    
        try {
          _dirtyElements[index].rebuild();
        } catch (e, stack) {
           ...   
        }
        index += 1;
        
      }
      
    } finally {
      
      _dirtyElements.clear();
      _scheduledFlushDirtyElements = false;
      _dirtyElementsNeedsResorting = null;
     
    }
}
```
还记得在请求 Vsync 之前把 dirty 的 Element 放进了 BuildOwner 的 _dirtyElements 数组里。因为父 Element 重建的时候其子节点也可能需要重建。父节点在前避免了重复的 build。

接下来就是 rebuild()
Element.rebuild()
```dart
void rebuild() {
    if (!_active || !_dirty)
      return;
    performRebuild();
    
    assert(!_dirty);
  }
```
Element 的 performRebuild()由其子类实现。我们本文的出发点是 State.setState() 那我们看下 StateFullElement 是怎么实现的。其实现在其父类 ComponentElement 里。
ComponentElement.performRebuild()
```dart
@override
  void performRebuild() {

    Widget built;
    try {
      built = build();
      debugWidgetBuilderValue(widget, built);
    } catch (e, stack) {

    } finally {
      _dirty = false;

    }
    try {
      _child = updateChild(_child, built, slot);

    } catch (e, stack) {
      _child = updateChild(null, built, slot);
    }

  }
```
这里的 build 最终会调用 State.build。返回的就是我们创建的 widget，然后去更新 updateChild，这个上篇文章里我们说过，更新规则如下：
|                     | **newWidget == null**  | **newWidget != null**   |
| :-----------------: | :--------------------- | :---------------------- |
|  **child == null**  |  Returns null.         |  Returns new [Element]. |
|  **child != null**  |  Old child is removed, returns null. | Old child updated if possible, returns child or new [Element]. |

我们这里是从 setState 过来，所以是更新，也就是说 child 的 update 会被调用。这个 update 在 RenderObjectElement 和普通 Element 里有些不同。
Element.update()
```dart
 void update(covariant Widget newWidget) {
    _widget = newWidget;
  }
```
RenderObjectElement.update()
```dart
 @override
  void update(covariant RenderObjectWidget newWidget) {
    super.update(newWidget);
    widget.updateRenderObject(this, renderObject);
    _dirty = false;
  }
```
updateRenderObject 也就是把最新的配置到 RenderObject 上。
StateFullElement.update()
```dart
 @override
  void update(StatefulWidget newWidget) {
    super.update(newWidget);
    final StatefulWidget oldWidget = _state._widget;
    
    rebuild();
  }
```
最后子节点又回进行 rebuild——>performRebuild()——>updateChild()——>child.update() 这样一直沿着 Element 树走下去。

至此 build 阶段已经跑完了，后续将继续分析渲染的其他阶段。

### 总结
* setState 会把当前的 Element 放入 BuildOwner 的 dirtyElements 数组里，然后请求 Vsync 事件。
* drawFrame 在 WidgetsBinding 里主要作 rebuild 也就是 widget 的重建。
* drawFrame 在 RenderBinding 里回进行 layout、paint 等操作，我们后续再分析。
* build 阶段会沿着被标为 dirty 的 Element 进行 rebuild。

