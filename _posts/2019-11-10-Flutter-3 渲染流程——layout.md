---
layout:     post
title:      Flutter-3 渲染流程——layout
date:       2019-11-10
author:     xflyme
header-img: img/post-bg-2019-11-10.jpeg
catalog: true
tags:
    - Flutter
    - 源码分析
---


### 前言
前一节 [Flutter-2 渲染流程——bulid](https://xfly.me/2019/11/09/Flutter-2-%E6%B8%B2%E6%9F%93%E6%B5%81%E7%A8%8B-bulid/) 我们分析了 Flutter 渲染流程中的第一个（严格来说不能是第一步，前面还有动画）步骤 build，今天来看一下 layout。


前文我们分析到，update 的时候回进行 updateRenderObject 也就是把最新的配置到 RenderObject 上。在更新的时候又会触发 markNeedsLayout(),本节我们就从 markNeedsLayout 开始。

### markNeedsLayout

RenderObject.markNeedsLayout()
```dart
void markNeedsLayout() {
    if (_needsLayout) {
      return;
    }
    if (_relayoutBoundary != this) {
      markParentNeedsLayout();
    } else {
      _needsLayout = true;
      if (owner != null) {
        owner._nodesNeedingLayout.add(this);
        owner.requestVisualUpdate();
      }
    }
  }
```

这里先做一个判断，如果已经 _needsLayout 直接返回。
然后判断 _relayoutBoundary，如果布局边界不是自己会沿着树向上把布局边界内的节点都加入 PipelineOwner 中的 _nodesNeedingLayout 数组，如果不是则需要对父元素进行 markNeedsLayout()。

markNeedsLayout 会回到 WidgetsBinding 的 drawFrame 方法。这里会进行 super 方法的调用，也就是我们昨天分析的 RenderBinding.drawFrame()
，由此开始就进入了我们今天分析的 layout 阶段。

### layout 阶段

RenderBinding.drawFrame
```dart
 @protected
  void drawFrame() {
    pipelineOwner.flushLayout();
    pipelineOwner.flushCompositingBits();
    pipelineOwner.flushPaint();
    renderView.compositeFrame(); // this sends the bits to the GPU
    pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
  }
```
第一个方法 pipelineOwner.flushLayout() 就是本节要分析的布局阶段。
```dart
/// Update the layout information for all dirty render objects.
  void flushLayout() {

    try {
      while (_nodesNeedingLayout.isNotEmpty) {
        final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
        _nodesNeedingLayout = <RenderObject>[];
        for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => a.depth - b.depth)) {
          if (node._needsLayout && node.owner == this)
            node._layoutWithoutResize();
        }
      }
    } finally {
    }
  }
```
注释里有说明这个方法的作用「为所有脏的 RenderOjbect 更新它们的布局信息」。在布局开始之前会对所有的 dirtyNodes 做一个排序，然后从上层节点开始调用节点的 node._layoutWithoutResize()。为什么从上层开始？布局的时候会沿着树向下进行，优先处理上层可以避免子节点的重复布局。

> 这里有一个问题，上层节点的布局是否一定会影响到下层节点？这和 RepaintBoundary 有关，我们后续再分析。

RenderObject._layoutWidhoutResize()
```dart
void _layoutWithoutResize() {

    try {
      performLayout();
      markNeedsSemanticsUpdate();
    } catch (e, stack) {
    }
    
    _needsLayout = false;
    markNeedsPaint();
  }
```
_layoutWithoutResize 中接着调用了 performLayout() 方法，然后 _neetsLayout 设为 true，在后边就是标记为需要 paint 即 markNeedsPaint()。
performLayout() 是一个空方法，需要子类自己实现。布局方式不同，它的布局逻辑也不同。这里我们先找一个比较简单的 RenderFlow 看一下它的 performLayout。

RenderFlow.performLayout()
```dart
 @override
  void performLayout() {
    size = _getSize(constraints);
    int i = 0;
    _randomAccessChildren.clear();
    RenderBox child = firstChild;
    while (child != null) {
      _randomAccessChildren.add(child);
      final BoxConstraints innerConstraints = _delegate.getConstraintsForChild(i, constraints);
      child.layout(innerConstraints, parentUsesSize: true);
      final FlowParentData childParentData = child.parentData;
      childParentData.offset = Offset.zero;
      child = childParentData.nextSibling;
      i += 1;
    }
  }
```
首先会根据 constraints 获取一个 size，说到这里我们先看一下 constraints 和 size 之间的关系。

#### constraints & size

其实布局的过程就是要确定各个元素的大小和位置，在这个过程中父元素（Android 中的 View 或 Flutter 中的 RenderObject）会将一定的约束（Contraints）传递给子元素。而子元素的 Size 可能会影响到父元素的 Size。整个过程就是约束向下传递，size 向上传递的过程，如图所示：

![图1](/img/flutter-3-1.png)

##### BoxConstraints
Flutter 中使用最多的是 BoxConstraints 即「盒子约束」，我们先简单说一下它的属性及特点。

![图2](/img/flutter-3-2.png)

它有四个属性分别是 minWidth、minHeight、maxWidth 和 maxHeight。
根据这四个属性又可以确定约束的 isTight、loose 等。
条件分别如下：
* tight 如果最小约束(minWidth，minHeight)和最大约束(maxWidth，maxHeight)分别都是一样的
* loose 如果最小约束都是0.0（不管最大约束），如果最小约束和最大约束都是0.0，就同时是tightly和loose
* bounded 如果最大约束都不是infinite
* unbounded 如果最大约束都是infinite
* expanding 如果最小约束和最大约束都是infinite

说完 constraints 我们继续回到 performLayout()

```dart
      child.layout(innerConstraints, parentUsesSize: true);
```

接下来是 layout() 方法，它是在 RenderObject 中实现的
```dart
void layout(Constraints constraints, { bool parentUsesSize = false }) {
    RenderObject relayoutBoundary;
    if (!parentUsesSize || sizedByParent || constraints.isTight || parent is! RenderObject) {
      relayoutBoundary = this;
    } else {
      final RenderObject parent = this.parent;
      relayoutBoundary = parent._relayoutBoundary;
    }

    if (!_needsLayout && constraints == _constraints && relayoutBoundary == _relayoutBoundary) {
      return;
    }
    _constraints = constraints;
    _relayoutBoundary = relayoutBoundary;

    if (sizedByParent) {
      try {
        performResize();
      } catch (e, stack) {
        ...
      }
    }
    try {
      performLayout();
      markNeedsSemanticsUpdate();
     
    } catch (e, stack) {
      ...
    }
    _needsLayout = false;
    markNeedsPaint();
  }

```
可以看一下注释，它做了很多重要的说明：
* 如果父元素需要子元素 layout 后的信息，parentUsesSize 应该设为 true，这种情况下每当子元素标记为需要 layout 的时候，父元素也会被标记为需要 layout。默认情况下 parentUsesSize 为 false 这种情况下子元素布局改变不需要通知父元素。
* 子元素不应该复写 layout 方法，可以腹泻 performResize 和 performLayout。layout 会把真正的工作分发到 performResize 和 performLayout 中执行。
* 父元素的 performLayout 应当无条件的调用所有孩子的 layout。如果孩子的布局信息无需改变的时候应当尽快的返回。

接下来看代码首先是 relayoutBoundary 的判定，它有几个条件：
* !parentUsesSize
* sizedByParent
* constraints.isTight
* parent is! RenderObject
以上条件满足其一的时候 relayoutBoundary 就是自己，否则 relayoutBoundary 是父元素的 relayoutBoundary。

接下来会做另一个判断：

```dart
 if (!_needsLayout && constraints == _constraints && relayoutBoundary == _relayoutBoundary) {
      return;
    }
```
如果当前节点不需要做重新布局，约束也没有变化，relayoutBoundary也没有变化就直接返回了。也就是说从这个节点开始，包括其下的子节点都不需要做重新布局了。

relayoutBoundary 对布局的影响有两点：
* 标脏的时候找到布局边界便停止向上标脏。
* 布局的时候如果布局边界没变（条件之一）停止向下继续布局。

然后是另一个判断，如果 sizedByParent 为 true，会调用 performResize()。这个函数会仅仅根据约束来计算当前 RenderObject 的尺寸。当这个函数被调用以后，通常接下来的 performLayout()函数里不能再更改尺寸了。

接下来又到了 performLayout()，layout() 和 performLayout() 会沿着需要 layout 的子树一直走下去。

至此，Flutter 的 layout 流程也已经分析完毕。

### 总结
* layout 过程是一个一下一上的过程，约束向下传递，size 向上传递。
* 子类不应该复写 layout，而应该复写 performResize 和 performLayout。
* perfromLayout 应该调用所有孩子的 layout，子元素布局信息不会改变的时候应该在 layout 中尽快返回。
* relayoutBoundary 防止对不必要的节点进行布局，提高布局效率。


