---
layout:     post
title:      Flutter-6 渲染流程——needsCompositing
date:       2019-11-12
author:     xflyme
header-img: img/post-bg-2019-11-12-2.jpeg
catalog: true
tags:
    - Flutter
    - 源码分析
---


### 前言
前面分别分析了 Flutter 渲染流程中的 build、layout 和 paint 等。还有一个阶段我跳了过去。
```dart
@protected
  void drawFrame() {
    assert(renderView != null);
    pipelineOwner.flushLayout();
    //这个阶段到底做什么
    pipelineOwner.flushCompositingBits();
    pipelineOwner.flushPaint();
    renderView.compositeFrame(); // this sends the bits to the GPU
    pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
  }
```
就是上面代码注释的这个阶段，我一直没弄明白这个阶段的目的。我们先看看代码，看它做了什么。

### needsCompositing

#### mark

先从 mark 阶段开始，代码如下：

```dart
void markNeedsCompositingBitsUpdate() {
    //1.已经被标记 直接返回
    if (_needsCompositingBitsUpdate)
      return;
    _needsCompositingBitsUpdate = true;
    if (parent is RenderObject) {
      final RenderObject parent = this.parent;
      if (parent._needsCompositingBitsUpdate)
        return;
      //2.向上标记 parent 直至 repaintBoundary
      if (!isRepaintBoundary && !parent.isRepaintBoundary) {
        parent.markNeedsCompositingBitsUpdate();
        return;
      }
    }
    if (owner != null)
      owner._nodesNeedingCompositingBitsUpdate.add(this);
  }
```
标记阶段主要是更新 _needsCompositingBitsUpdate 标志位，它不只会更新自己的也会更新 parent 的，直到自己或 parent 是一个 RepaintBoundary。
这段可以理解为**把同一个 layer 中当前节点的父节点的 _needsCompositingBitsUpdate 都更新为 true**。

同时这个方法的方法注释里有说明它自己一般被 adoptChild 或 dropChild 调用。

#### flushCompositingBits

mark 之后 RendererBinding.drawFrame() 会进行 flushCompositingBits，我们看一下后续阶段做了什么。

PipelineOwner.flushCompositingBits()
```dart
void flushCompositingBits() {
    //排序，深度浅的节点在前
    _nodesNeedingCompositingBitsUpdate.sort((RenderObject a, RenderObject b) => a.depth - b.depth);
    for (RenderObject node in _nodesNeedingCompositingBitsUpdate) {
      if (node._needsCompositingBitsUpdate && node.owner == this)
        node._updateCompositingBits();
    }
    //清空 _nodesNeedingCompositingBitsUpdate 数组
    _nodesNeedingCompositingBitsUpdate.clear();
  }
```

这个方法做了一个调度，调用了另一个方法 node._updateCompositingBits()。

node._updateCompositingBits()
```dart
void _updateCompositingBits() {
    if (!_needsCompositingBitsUpdate)
      return;
    final bool oldNeedsCompositing = _needsCompositing;
    _needsCompositing = false;
    visitChildren((RenderObject child) {
      child._updateCompositingBits();
      if (child.needsCompositing)
        _needsCompositing = true;
    });
    if (isRepaintBoundary || alwaysNeedsCompositing)
      _needsCompositing = true;
    if (oldNeedsCompositing != _needsCompositing)
      markNeedsPaint();
    _needsCompositingBitsUpdate = false;
  }
```
这个方法会遍历所有孩子节点，去更新它们 _needsCompositing 标志位。如果某个孩子 *isRepaintBoundary* 或 *alwaysNeedsCompositing* 这个孩子及她的所有 parent 的 _needsCompositing 都会被更新为 true。

还有一点需要注意的是，这里的 visitChildren 没有设置特殊退出条件，也就是说**当前节点的所有子孙节点都会被遍历**。

更新完 _needsCompositing 标志位后就看他用到哪了，答案是在 PaintingContext 中：

```dart
ClipRectLayer pushClipRect(bool needsCompositing, Offset offset, Rect clipRect, PaintingContextCallback painter, { 
    Clip clipBehavior = Clip.hardEdge, ClipRectLayer oldLayer }) {
    final Rect offsetClipRect = clipRect.shift(offset);
    if (needsCompositing) {
      final ClipRectLayer layer = oldLayer ?? ClipRectLayer();
      layer
        ..clipRect = offsetClipRect
        ..clipBehavior = clipBehavior;
      pushLayer(layer, painter, offset, childPaintBounds: offsetClipRect);
      return layer;
    } else {
      clipRectAndPaint(offsetClipRect, clipBehavior, offsetClipRect, () => painter(this, offset));
      return null;
    }
  }
  
  
ClipRRectLayer pushClipRRect(bool needsCompositing, Offset offset, Rect bounds, RRect clipRRect, PaintingContextCallback    painter, { Clip clipBehavior = Clip.antiAlias, ClipRRectLayer oldLayer }) {
    assert(clipBehavior != null);
    final Rect offsetBounds = bounds.shift(offset);
    final RRect offsetClipRRect = clipRRect.shift(offset);
    if (needsCompositing) {
      final ClipRRectLayer layer = oldLayer ?? ClipRRectLayer();
      layer
        ..clipRRect = offsetClipRRect
        ..clipBehavior = clipBehavior;
      pushLayer(layer, painter, offset, childPaintBounds: offsetBounds);
      return layer;
    } else {
      clipRRectAndPaint(offsetClipRRect, clipBehavior, offsetBounds, () => painter(this, offset));
      return null;
    }
  }
  
ClipPathLayer pushClipPath(bool needsCompositing, Offset offset, Rect bounds, Path clipPath, PaintingContextCallback painter, { Clip clipBehavior = Clip.antiAlias, ClipPathLayer oldLayer }) {
    assert(clipBehavior != null);
    final Rect offsetBounds = bounds.shift(offset);
    final Path offsetClipPath = clipPath.shift(offset);
    if (needsCompositing) {
      final ClipPathLayer layer = oldLayer ?? ClipPathLayer();
      layer
        ..clipPath = offsetClipPath
        ..clipBehavior = clipBehavior;
      pushLayer(layer, painter, offset, childPaintBounds: offsetBounds);
      return layer;
    } else {
      clipPathAndPaint(offsetClipPath, clipBehavior, offsetBounds, () => painter(this, offset));
      return null;
    }
  }
  
  
TransformLayer pushTransform(bool needsCompositing, Offset offset, Matrix4 transform, PaintingContextCallback painter, {    TransformLayer oldLayer }) {
    final Matrix4 effectiveTransform = Matrix4.translationValues(offset.dx, offset.dy, 0.0)
      ..multiply(transform)..translate(-offset.dx, -offset.dy);
    if (needsCompositing) {
      final TransformLayer layer = oldLayer ?? TransformLayer();
      layer.transform = effectiveTransform;
      pushLayer(
        layer,
        painter,
        offset,
        childPaintBounds: MatrixUtils.inverseTransformRect(effectiveTransform, estimatedBounds),
      );
      return layer;
    } else {
      canvas
        ..save()
        ..transform(effectiveTransform.storage);
      painter(this, offset);
      canvas
        ..restore();
      return null;
    }
  }
```

如果 needsCompositing 为 true 会 push 一个新的（不一定）图层。
而如果needsCompositing为false的话则走的是canvas的各种变换了。
**为什么为 true 的时候需要 push 一个图层？**

我现在还是没弄明白是为什么，大胆猜测一下：
> markNeedsCompositingBitsUpdate 一般会被 adoptChild 或 dropChild 调用，也就是有 RenderObject 添加或删除。这个时候可能会影响当前 layer 以及它的 child layer，所以需要 「Compositing」。

当然这只是一个猜测，如果有谁知道实际用途，烦请告诉我。
