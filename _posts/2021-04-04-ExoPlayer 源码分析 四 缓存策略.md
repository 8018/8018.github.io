---
layout:     post
title:      ExoPlayer 源码分析 四 缓存策略
date:       2021-04-04
author:     xflyme
header-img: img/post-bg-2021-04-04.jpg
catalog: true
tags:
    - 音视频
    - ExoPlayer
    - 源码分析
---


本文基于ExoPlayer 2.13.2 版。

ExoPlayer 通过 LoadControl 控制缓存：

```java
LoadControl loadControl = new DefaultLoadControl(new DefaultAllocator(BUFFER_SEGMENT_SIZE));

ChunkSampleSource videoSampleSource = new ChunkSampleSource(videoChunkSource, loadControl,
    VIDEO_BUFFER_SEGMENTS * BUFFER_SEGMENT_SIZE, mainHandler, player,
    DemoPlayer.TYPE_VIDEO);

ChunkSampleSource audioSampleSource = new ChunkSampleSource(audioChunkSource, loadControl,
    AUDIO_BUFFER_SEGMENTS * BUFFER_SEGMENT_SIZE, mainHandler, player,
    DemoPlayer.TYPE_AUDIO);
```


SampleSource enable 时会向 loadControl 注册，disable 时反注册。

LoadControl 真正控制是否继续缓存的方法是 update ，HlsSampleSource、ChunkSampleSource 都是通过这个方法确定是否继续缓存。

```java
DefaultLoadControl.update

@Override
public boolean update(Object loader, long playbackPositionUs, long nextLoadPositionUs,
    boolean loading) {
  // Update the loader state.
  int loaderBufferState = getLoaderBufferState(playbackPositionUs, nextLoadPositionUs);
  LoaderState loaderState = loaderStates.get(loader);
  boolean loaderStateChanged = loaderState.bufferState != loaderBufferState
      || loaderState.nextLoadPositionUs != nextLoadPositionUs || loaderState.loading != loading;
  if (loaderStateChanged) {
    loaderState.bufferState = loaderBufferState;
    loaderState.nextLoadPositionUs = nextLoadPositionUs;
    loaderState.loading = loading;
  }

  // Update the buffer state.
  int currentBufferSize = allocator.getTotalBytesAllocated();
  int bufferState = getBufferState(currentBufferSize);
  boolean bufferStateChanged = this.bufferState != bufferState;
  if (bufferStateChanged) {
    this.bufferState = bufferState;
  }

  // If either of the individual states have changed, update the shared control state.
  if (loaderStateChanged || bufferStateChanged) {
    updateControlState();
  }

  return nextLoadPositionUs != -1 && nextLoadPositionUs <= maxLoadStartPositionUs;
}

```

最终是通过计算 nextLoadPositionUs 是否小于 maxLoadStartPositionUs 且 nextLoadPositionUs 不等于 -1 来决定是否继续缓存。

```java
public static final int DEFAULT_LOW_WATERMARK_MS = 15000;
public static final int DEFAULT_HIGH_WATERMARK_MS = 30000;
public static final float DEFAULT_LOW_BUFFER_LOAD = 0.2f;
public static final float DEFAULT_HIGH_BUFFER_LOAD = 0.8f;

private static final int ABOVE_HIGH_WATERMARK = 0;
private static final int BETWEEN_WATERMARKS = 1;
private static final int BELOW_LOW_WATERMARK = 2;

```

LoadControl 在时间上和缓存占用比例上分别由两个「水位」：低水位和高水位，达到高水位时暂停缓存，达到低水位时继续缓存。
然后有两个方法 getLoaderBufferState 和 getBufferState 分别从时间和缓存数据大小上来计算缓存状态是否改变，如果改变则通过 updateControlState 更新 ControlState 。

```java
private int getLoaderBufferState(long playbackPositionUs, long nextLoadPositionUs) {
  if (nextLoadPositionUs == -1) {
    return ABOVE_HIGH_WATERMARK;
  } else {
    long timeUntilNextLoadPosition = nextLoadPositionUs - playbackPositionUs;
    return timeUntilNextLoadPosition > highWatermarkUs ? ABOVE_HIGH_WATERMARK :
        timeUntilNextLoadPosition < lowWatermarkUs ? BELOW_LOW_WATERMARK :
        BETWEEN_WATERMARKS;
  }
}

private int getBufferState(int currentBufferSize) {
  float bufferLoad = (float) currentBufferSize / targetBufferSize;
  return bufferLoad > highBufferLoad ? ABOVE_HIGH_WATERMARK
      : bufferLoad < lowBufferLoad ? BELOW_LOW_WATERMARK
      : BETWEEN_WATERMARKS;
}

```

```java
private void updateControlState() {
  boolean loading = false;
  boolean haveNextLoadPosition = false;
  int highestState = bufferState;
  for (int i = 0; i < loaders.size(); i++) {
    LoaderState loaderState = loaderStates.get(loaders.get(i));
    loading |= loaderState.loading;
    haveNextLoadPosition |= loaderState.nextLoadPositionUs != -1;
    highestState = Math.max(highestState, loaderState.bufferState);
  }

  fillingBuffers = !loaders.isEmpty() && (loading || haveNextLoadPosition)
      && (highestState == BELOW_LOW_WATERMARK
      || (highestState == BETWEEN_WATERMARKS && fillingBuffers));
  if (fillingBuffers && !streamingPrioritySet) {
    NetworkLock.instance.add(NetworkLock.STREAMING_PRIORITY);
    streamingPrioritySet = true;
    notifyLoadingChanged(true);
  } else if (!fillingBuffers && streamingPrioritySet && !loading) {
    NetworkLock.instance.remove(NetworkLock.STREAMING_PRIORITY);
    streamingPrioritySet = false;
    notifyLoadingChanged(false);
  }

  maxLoadStartPositionUs = -1;
  if (fillingBuffers) {
    for (int i = 0; i < loaders.size(); i++) {
      Object loader = loaders.get(i);
      LoaderState loaderState = loaderStates.get(loader);
      long loaderTime = loaderState.nextLoadPositionUs;
      if (loaderTime != -1
          && (maxLoadStartPositionUs == -1 || loaderTime < maxLoadStartPositionUs)) {
        maxLoadStartPositionUs = loaderTime;
      }
    }
  }
}

```

updateControlState 中真正起作用的是最后的 for 循环，其中会更新 maxLoadStartPositionUs 。

## 总结
LoadControl 中时间和缓存数据大小上分别有两个水位，低水位、高水位。
当水位变动的时候会更新 maxLoadStartPositionUs 来确定是否继续缓存——达到高水位暂停缓存，达到低水位开始缓存。