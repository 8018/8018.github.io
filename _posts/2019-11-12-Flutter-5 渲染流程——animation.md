---
layout:     post
title:      Flutter-5 渲染流程——animation
date:       2019-11-12
author:     xflyme
header-img: img/post-bg-2019-11-12.jpg
catalog: true
tags:
    - Flutter
    - 源码分析
---


动画我们简单说一下，只说它的原理，不跟踪代码。

### 回顾
我们先来回顾一下 Android 原生动画是怎么做的，无论补间动画或属性动画它都有这样几个属性：
* 开始状态（View 的属性或数值）
* 结束状态（View 的属性或数值）
* 持续时间
* 插值器（可选）

动画开始之后会进行以下步骤：
1. 根据开始时间、当前时间、持续时间算出动画的进度（一般是 0 到 1 之间的一个值）。
2. 根据进度用插值器计算出一个「插值后的值」（没想到更合适的描述）。
3. 最后根据插值器给出的值计算出一个 「Transformation」或一个属性应用到 view 上，然后需要重新执行渲染后续流程，参考下面的图。
4. 重复以上三个步骤直至动画完成。

![图1](/img/flutter-5-1.png)

![图2](/img/flutter-5-2.jpeg)

所以可以看到，无论是 Android、ios 还是 Flutter 都会把 Animation 放到 VSync 到来之后的第一个步骤。
