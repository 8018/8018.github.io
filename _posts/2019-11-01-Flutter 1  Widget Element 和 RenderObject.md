---
layout:     post
title:      Flutter 1 Widget Element 和 RenderObject
date:       2019-11-01
author:     xflyme
header-img: img/post-bg-2019-11-01.jpeg
catalog: true
tags:
    - Android
    - Flutter
    - RenderObjectElement
---


### 前言
和 Android 中 View 承载所有视图功能不同，Flutter 中控件系统涉及到三种不同的对象：Widget、Element 和 RenderObject，想要用好 Flutter 就必须搞清楚它们之间的关系。我们先来看一下他们之间的关系，后续在深入挖掘一下 Flutter 的整个渲染流程。

### 类定义
先看下这三个核心类的定义，看能找到哪些有用信息。
#### Widget
Widget 源码：
```dart
@immutable
abstract class Widget extends DiagnosticableTree {
  /// Initializes [key] for subclasses.
  const Widget({ this.key });

  
  final Key key;

  @protected
  Element createElement();

  @override
  String toStringShort() {
    return key == null ? '$runtimeType' : '$runtimeType-$key';
  }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.defaultDiagnosticsTreeStyle = DiagnosticsTreeStyle.dense;
  }

  static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
  }
}
```
其实 Widget 的类注释中包含很多有用的信息，可以好好读一下。本文就不把他们全部复制下来了，简单的翻译一下：
* Widget 表示 Element 的配置，Element 管理着底层的渲染树
* Widget 没有可变状态，所有的属性都必须是 final，如果想关联 State 和 Widget，请用 StateFullWidget
* 如果两个 widget 的 「key」和 「runtimeType」相等「==」，新的 widget 可以在底层的 element 树上替换旧的 widget。否则，会产生一个新的 element，并把新的 element 插入 element 树。
* 一个 widget 可以被放进树中多次。

现在我们可能无法理解注释中所有的意思，先放一边，后续再不时的回顾。

可以看到 widget 中有两个关键属性，key 和 Element。key 的作用注释中有解释，我们在继续看下 Element。

#### 
```dart
abstract class Element extends DiagnosticableTree implements BuildContext {
  /// Creates an element that uses the given widget as its configuration.
  ///
  /// Typically called by an override of [Widget.createElement].
  Element(Widget widget)
    : assert(widget != null),
      _widget = widget;

  // parent 不解释
  Element _parent;

  ...

 
  dynamic get slot => _slot;
  dynamic _slot;

  int get depth => _depth;
  int _depth;


  /// The configuration for this element.
  @override
  Widget get widget => _widget;
  Widget _widget;

  /// The object that manages the lifecycle of this element.
  @override
  BuildOwner get owner => _owner;
  BuildOwner _owner;

  bool _active = false;

  

  /// 当前位置或当前位置以下的 RenderObject
  /// 如果当前 Element 是 RenderObjectElement 返回
  /// 如果当前 Element 不时 RenderObjectElement，向子树中继续寻找
  RenderObject get renderObject {
    RenderObject result;
    void visit(Element element) {
      assert(result == null); // this verifies that there's only one child
      if (element is RenderObjectElement)
        result = element.renderObject;
      else
        element.visitChildren(visit);
    }
    visit(this);
    return result;
  }

 
  @override
  void visitChildElements(ElementVisitor visitor) {
    assert(() {
      if (owner == null || !owner._debugStateLocked)
        return true;
      throw FlutterError.fromParts(<DiagnosticsNode>[
        ErrorSummary('visitChildElements() called during build.'),
        ErrorDescription(
          'The BuildContext.visitChildElements() method can\'t be called during '
          'build because the child list is still being updated at that point, '
          'so the children might not be constructed yet, or might be old children '
          'that are going to be replaced.'
        )
      ]);
    }());
    visitChildren(visitor);
  }

  /// Update the given child with the given new configuration.
  ///
  /// This method is the core of the widgets system. It is called each time we
  /// are to add, update, or remove a child based on an updated configuration.
  ///
  /// If the `child` is null, and the `newWidget` is not null, then we have a new
  /// child for which we need to create an [Element], configured with `newWidget`.
  ///
  /// If the `newWidget` is null, and the `child` is not null, then we need to
  /// remove it because it no longer has a configuration.
  ///
  /// If neither are null, then we need to update the `child`'s configuration to
  /// be the new configuration given by `newWidget`. If `newWidget` can be given
  /// to the existing child (as determined by [Widget.canUpdate]), then it is so
  /// given. Otherwise, the old child needs to be disposed and a new child
  /// created for the new configuration.
  ///
  /// If both are null, then we don't have a child and won't have a child, so we
  /// do nothing.
  ///
  /// The [updateChild] method returns the new child, if it had to create one,
  /// or the child that was passed in, if it just had to update the child, or
  /// null, if it removed the child and did not replace it.
  ///
  /// The following table summarizes the above:
  ///
  /// |                     | **newWidget == null**  | **newWidget != null**   |
  /// | :-----------------: | :--------------------- | :---------------------- |
  /// |  **child == null**  |  Returns null.         |  Returns new [Element]. |
  /// |  **child != null**  |  Old child is removed, returns null. | Old child updated if possible, returns child or new [Element]. |
  @protected
  Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
    assert(() {
      if (newWidget != null && newWidget.key is GlobalKey) {
        final GlobalKey key = newWidget.key;
        key._debugReserveFor(this);
      }
      return true;
    }());
    if (newWidget == null) {
      if (child != null)
        deactivateChild(child);
      return null;
    }
    if (child != null) {
      if (child.widget == newWidget) {
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        return child;
      }
      if (Widget.canUpdate(child.widget, newWidget)) {
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        child.update(newWidget);
        assert(child.widget == newWidget);
        assert(() {
          child.owner._debugElementWasRebuilt(child);
          return true;
        }());
        return child;
      }
      deactivateChild(child);
      assert(child._parent == null);
    }
    return inflateWidget(newWidget, newSlot);
  }

  /// Add this element to the tree in the given slot of the given parent.
  ///
  /// The framework calls this function when a newly created element is added to
  /// the tree for the first time. Use this method to initialize state that
  /// depends on having a parent. State that is independent of the parent can
  /// more easily be initialized in the constructor.
  ///
  /// This method transitions the element from the "initial" lifecycle state to
  /// the "active" lifecycle state.
  @mustCallSuper
  void mount(Element parent, dynamic newSlot) {
    ...
    _parent = parent;
    _slot = newSlot;
    _depth = _parent != null ? _parent.depth + 1 : 1;
    _active = true;
    if (parent != null) // Only assign ownership if the parent is non-null
      _owner = parent.owner;
    if (widget.key is GlobalKey) {
      final GlobalKey key = widget.key;
      key._register(this);
    }
    _updateInheritance();
    assert(() { _debugLifecycleState = _ElementLifecycle.active; return true; }());
  }

  /// Change the widget used to configure this element.
  ///
  /// The framework calls this function when the parent wishes to use a
  /// different widget to configure this element. The new widget is guaranteed
  /// to have the same [runtimeType] as the old widget.
  ///
  /// This function is called only during the "active" lifecycle state.
  @mustCallSuper
  void update(covariant Widget newWidget) {
    // This code is hot when hot reloading, so we try to
    // only call _AssertionError._evaluateAssertion once.
    assert(_debugLifecycleState == _ElementLifecycle.active
        && widget != null
        && newWidget != null
        && newWidget != widget
        && depth != null
        && _active
        && Widget.canUpdate(widget, newWidget));
    _widget = newWidget;
  }

  void _updateDepth(int parentDepth) {
    final int expectedDepth = parentDepth + 1;
    if (_depth < expectedDepth) {
      _depth = expectedDepth;
      visitChildren((Element child) {
        child._updateDepth(expectedDepth);
      });
    }
  }

  /// Remove [renderObject] from the render tree.
  ///
  /// The default implementation of this function simply calls
  /// [detachRenderObject] recursively on its child. The
  /// [RenderObjectElement.detachRenderObject] override does the actual work of
  /// removing [renderObject] from the render tree.
  ///
  /// This is called by [deactivateChild].
  void detachRenderObject() {
    visitChildren((Element child) {
      child.detachRenderObject();
    });
    _slot = null;
  }

  /// Add [renderObject] to the render tree at the location specified by [slot].
  ///
  /// The default implementation of this function simply calls
  /// [attachRenderObject] recursively on its child. The
  /// [RenderObjectElement.attachRenderObject] override does the actual work of
  /// adding [renderObject] to the render tree.
  void attachRenderObject(dynamic newSlot) {
    assert(_slot == null);
    visitChildren((Element child) {
      child.attachRenderObject(newSlot);
    });
    _slot = newSlot;
  }

 

  /// Create an element for the given widget and add it as a child of this
  /// element in the given slot.
  ///
  /// This method is typically called by [updateChild] but can be called
  /// directly by subclasses that need finer-grained control over creating
  /// elements.
  ///
  /// If the given widget has a global key and an element already exists that
  /// has a widget with that global key, this function will reuse that element
  /// (potentially grafting it from another location in the tree or reactivating
  /// it from the list of inactive elements) rather than creating a new element.
  ///
  /// The element returned by this function will already have been mounted and
  /// will be in the "active" lifecycle state.
  @protected
  Element inflateWidget(Widget newWidget, dynamic newSlot) {
    assert(newWidget != null);
    final Key key = newWidget.key;
    if (key is GlobalKey) {
      final Element newChild = _retakeInactiveElement(key, newWidget);
      if (newChild != null) {
        assert(newChild._parent == null);
        assert(() { _debugCheckForCycles(newChild); return true; }());
        newChild._activateWithParent(this, newSlot);
        final Element updatedChild = updateChild(newChild, newWidget, newSlot);
        assert(newChild == updatedChild);
        return updatedChild;
      }
    }
    final Element newChild = newWidget.createElement();
    assert(() { _debugCheckForCycles(newChild); return true; }());
    newChild.mount(this, newSlot);
    assert(newChild._debugLifecycleState == _ElementLifecycle.active);
    return newChild;
  }

  

  /// Marks the element as dirty and adds it to the global list of widgets to
  /// rebuild in the next frame.
  ///
  /// Since it is inefficient to build an element twice in one frame,
  /// applications and widgets should be structured so as to only mark
  /// widgets dirty during event handlers before the frame begins, not during
  /// the build itself.
  void markNeedsBuild() {

    if (!_active)
      return;
    ...
    if (dirty)
      return;
    _dirty = true;
    owner.scheduleBuildFor(this);
  }

  /// Called by the [BuildOwner] when [BuildOwner.scheduleBuildFor] has been
  /// called to mark this element dirty, by [mount] when the element is first
  /// built, and by [update] when the widget has changed.
  void rebuild() {
    assert(_debugLifecycleState != _ElementLifecycle.initial);
    if (!_active || !_dirty)
      return;
    ...
    performRebuild();
    ...
  }

  /// Called by rebuild() after the appropriate checks have been made.
  @protected
  void performRebuild();
}
```
可以看到 Element 实现了 BuildContext 接口，从名字上来看它是用来构建的上下文，我们以后再分析它。

Element 源码比 Widget 多很多，我们本节的目标是理清它们三个之间的关系，不深入具体的逻辑。我们先删掉大部分代码，保留一些重点属性和方法。
同样，先看注释：
* Element 表示 Widget 在树种的具体表现。
* 大部分 element 只有一个孩子，有些 element 比如 「RenderObjectElement」可以有多个孩子。
* Element 有以下生命周期：initial、active、inactive

Element 中有大量注释，并不是很难看懂。其中有一个重要方法就是 updateChild，它的目标是根据新的配置（Widget）更新 child。它的更新策略如下，注释解释也很详细，就不一一翻译了。它的 update 策略如下：

|                     | **newWidget == null**  | **newWidget != null**   |
| :-----------------: | :--------------------- | :---------------------- |
|  **child == null**  |  Returns null.         |  Returns new [Element]. |
|  **child != null**  |  Old child is removed, returns null. | Old child updated if possible, returns child or new 

另外，Element 中还有跟 RenderObject 相关的一些操作，其中 get 方法涉及到了另外一个类 RenderObjectElement，我们看一下。

#### RenderObjectElement

```dart
abstract class RenderObjectElement extends Element {
  /// Creates an element that uses the given widget as its configuration.
  RenderObjectElement(RenderObjectWidget widget) : super(widget);

  @override
  RenderObjectWidget get widget => super.widget;

  /// The underlying [RenderObject] for this element.
  @override
  RenderObject get renderObject => _renderObject;
  RenderObject _renderObject;

  RenderObjectElement _ancestorRenderObjectElement;

  // 找到祖先是 RenderObjectElement 的节点
  RenderObjectElement _findAncestorRenderObjectElement() {
    Element ancestor = _parent;
    while (ancestor != null && ancestor is! RenderObjectElement)
      ancestor = ancestor._parent;
    return ancestor;
  }

  ParentDataElement<RenderObjectWidget> _findAncestorParentDataElement() {
    Element ancestor = _parent;
    while (ancestor != null && ancestor is! RenderObjectElement) {
      if (ancestor is ParentDataElement<RenderObjectWidget>)
        return ancestor;
      ancestor = ancestor._parent;
    }
    return null;
  }

 ...
}
```
RenderObjectElement 故名思义它应该跟 RenderObject 有关，果不其然，它有两个涉及到 RenderObject 的成员：RenderObjectWidget 和 RenderObject。
其中 RenderObject 是在 mount 方法中被 RenderObjectWidget 创建：
```dart
@override
  void mount(Element parent, dynamic newSlot) {
    ...
    _renderObject = widget.createRenderObject(this);
    ...
  }
```
另外还有一个方法 _findAncestorRenderObjectElement ，这个方法是沿着 Element 树向上寻找祖先中的 RenderObjectElement 节点。 
当找到上面的第一个 RenderObjectElement 调用 attachRenderObject 的让他们产生关联。

```dart
  @override
  void attachRenderObject(dynamic newSlot) {
    assert(_ancestorRenderObjectElement == null);
    _slot = newSlot;
    _ancestorRenderObjectElement = _findAncestorRenderObjectElement();
    _ancestorRenderObjectElement?.insertChildRenderObject(renderObject, newSlot);
    final ParentDataElement<RenderObjectWidget> parentDataElement = _findAncestorParentDataElement();
    if (parentDataElement != null)
      _updateParentData(parentDataElement.widget);
  }
```

本文的目标到这里就已经差不多了，它们三个的关系已经比较明显。RenderObject 的代码量非常庞大，后续分析 layout 和渲染的时候再看。

#### RenderObject
待续

### 总结
Element 树和 RenderObject 树中节点不是一一对应的，其中的关键是 RenderObjectElement 和普通 Element。
RenderObjectElement 包含 RenderObject，而 Element 树在 RenderObject 树构建时起到很大的作用。因为，RenderObjectElement mount 的时候会沿着 Element 树向上寻找祖先 RenderObjectElement 节点并绑定。

