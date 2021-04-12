---
layout:     post
title:      ExoPlayer 源码分析 五 码率自适应
date:       2021-04-05
author:     xflyme
header-img: img/post-bg-2021-04-05.jpg
catalog: true
tags:
    - 音视频
    - ExoPlayer
    - 源码分析
---


本文基于 ExoPlayer 2.13.2 版。

## 带宽预估

DASH 和 Hls 都可以提供不同码率的流，ExoPlayer 使用 BandwidthMeter 来计算带宽，它的默认实现是 DefaultBandwidthMeter 。

```java
public interface BandwidthMeter extends TransferListener {

  public interface EventListener {

    /**
     * Invoked periodically to indicate that bytes have been transferred.
     */
    void onBandwidthSample(int elapsedMs, long bytes, long bitrate);
  }

  /**
   * Indicates no bandwidth estimate is available.
   */
  final long NO_ESTIMATE = -1;

  /**
   * Gets the estimated bandwidth, in bits/sec.
   *
   * @return Estimated bandwidth in bits/sec, or {@link #NO_ESTIMATE} if no estimate is available.
   */
  long getBitrateEstimate();

}

```


```java
public final class DefaultBandwidthMeter implements BandwidthMeter {

  public static final int DEFAULT_MAX_WEIGHT = 2000;

  private final Handler eventHandler;
  private final EventListener eventListener;
  private final Clock clock;
  // 计算码率的工具，使用 moving average 来计算
  private final SlidingPercentile slidingPercentile;
  // 一个计算周期接受的字节数
  private long bytesAccumulator;
  // 一个计算周期的开始时间
  private long startTimeMs;
  // 带宽估值
  private long bitrateEstimate;
  private int streamCount;

  public DefaultBandwidthMeter() {
    this(null, null);
  }

  public DefaultBandwidthMeter(Handler eventHandler, EventListener eventListener) {
    this(eventHandler, eventListener, new SystemClock());
  }

  public DefaultBandwidthMeter(Handler eventHandler, EventListener eventListener, Clock clock) {
    this(eventHandler, eventListener, clock, DEFAULT_MAX_WEIGHT);
  }

  public DefaultBandwidthMeter(Handler eventHandler, EventListener eventListener, int maxWeight) {
    this(eventHandler, eventListener, new SystemClock(), maxWeight);
  }

  public DefaultBandwidthMeter(Handler eventHandler, EventListener eventListener, Clock clock,
      int maxWeight) {
    this.eventHandler = eventHandler;
    this.eventListener = eventListener;
    this.clock = clock;
    this.slidingPercentile = new SlidingPercentile(maxWeight);
    bitrateEstimate = NO_ESTIMATE;
  }

  @Override
  public synchronized long getBitrateEstimate() {
    return bitrateEstimate;
  }

  @Override
  public synchronized void onTransferStart() {
    if (streamCount == 0) {
      startTimeMs = clock.elapsedRealtime();
    }
    streamCount++;
  }

  @Override
  public synchronized void onBytesTransferred(int bytes) {
    bytesAccumulator += bytes;
  }

  @Override
  public synchronized void onTransferEnd() {
    Assertions.checkState(streamCount > 0);
    long nowMs = clock.elapsedRealtime();
    int elapsedMs = (int) (nowMs - startTimeMs);
    if (elapsedMs > 0) {
      // 单位用的是字节和毫秒所以要 *8000
      float bitsPerSecond = (bytesAccumulator * 8000) / elapsedMs;
      slidingPercentile.addSample((int) Math.sqrt(bytesAccumulator), bitsPerSecond);
      // 使用 slidingPercentile.getPercentile 计算出带宽估值
      float bandwidthEstimateFloat = slidingPercentile.getPercentile(0.5f);
      bitrateEstimate = Float.isNaN(bandwidthEstimateFloat) ? NO_ESTIMATE
          : (long) bandwidthEstimateFloat;
      notifyBandwidthSample(elapsedMs, bytesAccumulator, bitrateEstimate);
    }
    streamCount--;
    if (streamCount > 0) {
      startTimeMs = nowMs;
    }
    bytesAccumulator = 0;
  }

  private void notifyBandwidthSample(final int elapsedMs, final long bytes, final long bitrate) {
    if (eventHandler != null && eventListener != null) {
      eventHandler.post(new Runnable()  {
        @Override
        public void run() {
          eventListener.onBandwidthSample(elapsedMs, bytes, bitrate);
        }
      });
    }
  }

}

```

slidingPercentile.addSample((int) Math./sqrt/(bytesAccumulator), bitsPerSecond) 会将当前采样周期传输的字节开方以及传输速率封装成一个 Sample 放入 SlidingPercentile 中的集合里。

向集合中添加 Sample 的时候会保证当前所有 Sample 的总 weight 保持在设定的 maxWeight 以下，如果超出则移除最早添加的数据。

带宽的预估则采用如下方法：

```java
/**
 * Compute the percentile by integration.
 *
 * @param percentile The desired percentile, expressed as a fraction in the range (0,1].
 * @return The requested percentile value or Float.NaN.
 */
public float getPercentile(float percentile) {
  // Sample 按 value 从小到大排列
  ensureSortedByValue();
  float desiredWeight = percentile * totalWeight;
  int accumulatedWeight = 0;
  for (int i = 0; i < samples.size(); i++) {
    Sample currentSample = samples.get(i);
    // weight 相加
    accumulatedWeight += currentSample.weight;
   // 向加后的 weight 达到 desiredWeight 时返回当前 sample 的value
    if (accumulatedWeight >= desiredWeight) {
      return currentSample.value;
    }
  }
  // Clamp to maximum value or NaN if no values.
  return samples.isEmpty() ? Float.NaN : samples.get(samples.size() - 1).value;
}

```

## Hls 码率切换
### 切换时机 & 条件
Hls 码率切换的代码在 HlsChunkSource#getChunkOperation 中，它首先会通过 getNextVariantIndex 获得一个候选 Variant id，其主要步骤及条件如下：
* 根据预估带宽获取候选 Variant id 即 idealIndex。
* 切向码率更低的流，缓存数据的可用时间要小于 20 s。
* 切向码率更高的流，缓存数据的可用时间要大于 5 s。

得到 Variant index 之后会回到 getChunkOperation 代码如下：

```java
public void getChunkOperation(TsChunk previousTsChunk, long playbackPositionUs,
    ChunkOperationHolder out) {
  int previousChunkVariantIndex =
      previousTsChunk == null ? -1 : getVariantIndex(previousTsChunk.format);
  // 下一个 VariantIndex
  int nextVariantIndex = getNextVariantIndex(previousTsChunk, playbackPositionUs);
  boolean switchingVariant = previousTsChunk != null
      && previousChunkVariantIndex != nextVariantIndex;

  HlsMediaPlaylist mediaPlaylist = variantPlaylists[nextVariantIndex];
  if (mediaPlaylist == null) {
    // We don't have the media playlist for the next variant. Request it now.
    // 当前不存在特定的 media playlist 需要请求
    out.chunk = newMediaPlaylistChunk(nextVariantIndex);
    return;
  }

  selectedVariantIndex = nextVariantIndex;
  int chunkMediaSequence;
  // 根据 live 与否，计算下一个 Chunk 的 MediaSequence
  if (live) {
    if (previousTsChunk == null) {
      chunkMediaSequence = getLiveStartChunkSequenceNumber(selectedVariantIndex);
    } else {
      chunkMediaSequence = getLiveNextChunkSequenceNumber(previousTsChunk.chunkIndex,
          previousChunkVariantIndex, selectedVariantIndex);
      if (chunkMediaSequence < mediaPlaylist.mediaSequence) {
        fatalError = new BehindLiveWindowException();
        return;
      }
    }
  } else {
    // Not live.
    if (previousTsChunk == null) {
      chunkMediaSequence = Util.binarySearchFloor(mediaPlaylist.segments, playbackPositionUs,
          true, true) + mediaPlaylist.mediaSequence;
    } else if (switchingVariant) {
      chunkMediaSequence = Util.binarySearchFloor(mediaPlaylist.segments,
          previousTsChunk.startTimeUs, true, true) + mediaPlaylist.mediaSequence;
    } else {
      chunkMediaSequence = previousTsChunk.getNextChunkIndex();
    }
  }

  // 
  int chunkIndex = chunkMediaSequence - mediaPlaylist.mediaSequence;
  if (chunkIndex >= mediaPlaylist.segments.size()) {
    if (!mediaPlaylist.live) {
      out.endOfStream = true;
    } else if (shouldRerequestLiveMediaPlaylist(selectedVariantIndex)) {
      out.chunk = newMediaPlaylistChunk(selectedVariantIndex);
    }
    return;
  }

  ...

}


```

### 缓存切换

Hls 的 RollingSampleBuffer 缓存在 DefaultTrackOutput 中，码率切换时会创建新的 DefaultTrackOutput，解码时只需要选一个合适的时机切换 DefaultTrackOutput就好了。
具体做法是：
* DefaultTrackOutput 中有个变量：spliceOutTimeUs 控制当前缓存需要切出时间。
* 当有多个 HlsExtractorWrapper 时会计算这个时间。
* 时间到了切换到下一个 HlsExtractorWrapper

```java
DefaultTrackOutput. configureSpliceTo

public boolean configureSpliceTo(DefaultTrackOutput nextQueue) {
  if (spliceOutTimeUs != Long.MIN_VALUE) {
    // We've already configured the splice.
    return true;
  }
  long firstPossibleSpliceTime;
  if (rollingBuffer.peekSample(sampleInfoHolder)) {
    firstPossibleSpliceTime = sampleInfoHolder.timeUs;
  } else {
    firstPossibleSpliceTime = lastReadTimeUs + 1;
  }
  RollingSampleBuffer nextRollingBuffer = nextQueue.rollingBuffer;
  while (nextRollingBuffer.peekSample(sampleInfoHolder)
      && (sampleInfoHolder.timeUs < firstPossibleSpliceTime || !sampleInfoHolder.isSyncFrame())) {
    // Discard samples from the next queue for as long as they are before the earliest possible
    // splice time, or not keyframes.
    nextRollingBuffer.skipSample();
  }
  if (nextRollingBuffer.peekSample(sampleInfoHolder)) {
    // We've found a keyframe in the next queue that can serve as the splice point. Set the
    // splice point now.
    spliceOutTimeUs = sampleInfoHolder.timeUs;
    return true;
  }
  return false;
}

```

```java

private boolean advanceToEligibleSample() {
  boolean haveNext = rollingBuffer.peekSample(sampleInfoHolder);
  if (needKeyframe) {
    while (haveNext && !sampleInfoHolder.isSyncFrame()) {
      rollingBuffer.skipSample();
      haveNext = rollingBuffer.peekSample(sampleInfoHolder);
    }
  }
  if (!haveNext) {
    return false;
  }
  // 如果sampleInfoHolder.timeUs 超过了spliceOutTimeUs 返回 false
  if (spliceOutTimeUs != Long.MIN_VALUE && sampleInfoHolder.timeUs >= spliceOutTimeUs) {
    return false;
  }
  return true;
}

public boolean isEmpty() {
  return !advanceToEligibleSample();
}
```


读取数据的时候会判断缓存是否为空，这时候就用到了上面计算的 spliceOutTimeUs 。

## DASH 码率切换
### 切换时机
切换代码在 FormatEvaluator 中，代码如下：

```java
@Override
public void evaluate(List<? extends MediaChunk> queue, long playbackPositionUs,
    Format[] formats, Evaluation evaluation) {
  long bufferedDurationUs = queue.isEmpty() ? 0
      : queue.get(queue.size() - 1).endTimeUs - playbackPositionUs;
  Format current = evaluation.format;
  // 根据带宽选择一个合适的 Format
  Format ideal = determineIdealFormat(formats, bandwidthMeter.getBitrateEstimate());
  // 是否码率提升
  boolean isHigher = ideal != null && current != null && ideal.bitrate > current.bitrate;
  // 是否码率降低
  boolean isLower = ideal != null && current != null && ideal.bitrate < current.bitrate;
  if (isHigher) {
    // 码率提升但是当前缓存小于 10s，放弃
    if (bufferedDurationUs < minDurationForQualityIncreaseUs) {
      // The ideal format is a higher quality, but we have insufficient buffer to
      // safely switch up. Defer switching up for now.
      ideal = current;
    } else if (bufferedDurationUs >= minDurationToRetainAfterDiscardUs) {
      // We're switching from an SD stream to a stream of higher resolution. Consider
      // discarding already buffered media chunks. Specifically, discard media chunks starting
      // from the first one that is of lower bandwidth, lower resolution and that is not HD.
      for (int i = 1; i < queue.size(); i++) {
        MediaChunk thisChunk = queue.get(i);
        long durationBeforeThisSegmentUs = thisChunk.startTimeUs - playbackPositionUs;
        if (durationBeforeThisSegmentUs >= minDurationToRetainAfterDiscardUs
            && thisChunk.format.bitrate < ideal.bitrate
            && thisChunk.format.height < ideal.height
            && thisChunk.format.height < 720
            && thisChunk.format.width < 1280) {
          // Discard chunks from this one onwards.
          evaluation.queueSize = i;
          break;
        }
      }
    }
  } else if (isLower && current != null
    && bufferedDurationUs >= maxDurationForQualityDecreaseUs) {
    // 向低码率转换，但是当前缓存足够，先不转
    // The ideal format is a lower quality, but we have sufficient buffer to defer switching
    // down for now.
    ideal = current;
  }
  if (current != null && ideal != current) {
    evaluation.trigger = Chunk.TRIGGER_ADAPTIVE;
  }
  evaluation.format = ideal;
}

```

#### 小结
* 向高码率切换，当前缓存需要大于 10s。
* 向低码率切换，当前缓存需要小于 25s。

### 缓存切换
与 Hls 会使用多个 DefaultTrackOutput 来缓存数据不同，DASH 仅使用一个（音频、视频、字幕流各有一个） DefaultTrackOutput 来保存数据。在码率切换的时候如果是码率提升切已缓存数据大于 25 s，那就要丢掉一部分数据然后加载更高码率的数据。
这个做法需要三步：

* 计算出从哪块开始加载新的数据
* 这块之后的缓存数据丢弃。
* 从这块开始加载更高码率的数据

代码中记录这「块」的索引的变量是 queueSize。

1、计算在上面贴出 FormatEvaluator evaluate 方法中，这里再贴一遍。

```java
    // 码率提升但是当前缓存小于 10s，放弃
    if (bufferedDurationUs < minDurationForQualityIncreaseUs) {
      // The ideal format is a higher quality, but we have insufficient buffer to
      // safely switch up. Defer switching up for now.
      ideal = current;
    } else if (bufferedDurationUs >= minDurationToRetainAfterDiscardUs) {
      // 如果已经缓存的数据很多，这时候我们只要保留一部分数据来满足当前观看，然后缓存更高质量的数据
		// minDurationToRetainAfterDiscardUs（25s） 是一个时间，只需要保留这段时间的数据，后边的丢弃
      // We're switching from an SD stream to a stream of higher resolution. Consider
      // discarding already buffered media chunks. Specifically, discard media chunks starting
      // from the first one that is of lower bandwidth, lower resolution and that is not HD.
      for (int i = 1; i < queue.size(); i++) {
        MediaChunk thisChunk = queue.get(i);
        long durationBeforeThisSegmentUs = thisChunk.startTimeUs - playbackPositionUs;
        // 根据 minDurationToRetainAfterDiscardUs 计算从哪块开始丢弃
        if (durationBeforeThisSegmentUs >= minDurationToRetainAfterDiscardUs
            && thisChunk.format.bitrate < ideal.bitrate
            && thisChunk.format.height < ideal.height
            && thisChunk.format.height < 720
            && thisChunk.format.width < 1280) {
          // Discard chunks from this one onwards.
          evaluation.queueSize = i;
          break;
        }
      }
    }
  }
```

2、丢数据代码：

```java
private boolean discardUpstreamMediaChunks(int queueLength) {
  if (mediaChunks.size() <= queueLength) {
    return false;
  }
  long startTimeUs = 0;
  long endTimeUs = mediaChunks.getLast().endTimeUs;

  BaseMediaChunk removed = null;
  while (mediaChunks.size() > queueLength) {
    removed = mediaChunks.removeLast();
    startTimeUs = removed.startTimeUs;
    loadingFinished = false;
  }
  // getFirstSampleIndex 是当前 chunk 缓存的第一个 sample 的索引
  // 从这个索引往后的数据都要丢掉
  sampleQueue.discardUpstreamSamples(removed.getFirstSampleIndex());

  notifyUpstreamDiscarded(startTimeUs, endTimeUs);
  return true;
}

```

3、构建新的 Chunk，重新加载数据。

```java
out.queueSize = evaluation.queueSize;

MediaChunk previous = queue.get(out.queueSize - 1);
long nextSegmentStartTimeUs = previous.endTimeUs;

int segmentNum = queue.isEmpty() ? representationHolder.getSegmentNum(playbackPositionUs)
      : startingNewPeriod ? representationHolder.getFirstAvailableSegmentNum()
      : queue.get(out.queueSize - 1).getNextChunkIndex();

Chunk nextMediaChunk = newMediaChunk(periodHolder, representationHolder, dataSource,
    mediaFormat, enabledTrack, segmentNum, evaluation.trigger, mediaFormat != null);
lastChunkWasInitialization = false;
out.chunk = nextMediaChunk;

```

如果切向低码率的流，第二步是不需要的。

### 小结

* DASH 一个视频流的数据缓存在一个，RollingBuffer 里。
* 码率切换就是选择一个时间点，在这个时间之后缓存改变码率后的数据。
* 如果是码率提升且已缓存数据可播放时间大于 25s，需要把 25s 以后的缓存丢弃，然后缓存更高码率的数据。






















