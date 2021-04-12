---
layout:     post
title:      ExoPlayer 源码分析 三 变速播放
date:       2021-04-03
author:     xflyme
header-img: img/post-bg-2021-04-03.jpeg
catalog: true
tags:
    - 音视频
    - ExoPlayer
    - 源码分析
---


本文基于 ExoPlayer 2.13.2 版。

## 变速播放
### 视频变速
音视频同步以音频为主时钟的情况下，视频变速基本上不需要做任何处理，这点不理解的可以看我 [ijkplayer 源码分析](/2021/03/27/ijkplayer-源码分析/) 中倍速播放的部分。

### 音频变速
音频变速主要有三种方式：
* 使用 soundtouch 处理音频，实现变速。
* 使用 sonic 处理音频，实现变速。
* 直接给 AudioTrack 设置  PlaybackParams 实现变速。

ExoPlayer 2.13.2 中使用的是第三种。