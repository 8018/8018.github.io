---
layout:     post
title:      ExoPlayer 源码分析 一 HLS 拉流及播放流程分析
date:       2021-04-01
author:     xflyme
header-img: img/post-bg-2021-04-01.jpg
catalog: true
tags:
    - 音视频
    - ExoPlayer
    - 源码分析
---


本文基于 ExoPlayer 2.13.2 版。

本文将从 HLS 入手快速的分析一下 ExoPlayer 各个组件的作用以及 HLS 从拉流到播放的整个流程。

## HLS 拉流播放步骤
1. 下载并解析 m3u8 文件内容
2. 拉流
2.1 加载视频流
2.2 加载音频流（如果有）
3. 解封装
4. 解码
4.1 视频解码
4.2 音频解码
5. 同步播放

先把整体流程列出来是为了带着问题及目标去分析源码，做到有的放矢，下面将一步步具体分析。

## 下载并解析 m3u8 文件 
简单而言，m3u8 文件分为两种格式：
* 内容直接给出 TS 文件索引
* 音频和视频分开，内容为不同码率的音频和视频的 m3u8 索引

以上两种分法是错的，具体请看[官方文档](https://developer.apple.com/library/archive/technotes/tn2288/_index.html)

两种文件的内容如下：
```
#EXTM3U
#EXT-X-TARGETDURATION:10
#EXT-X-VERSION:3
#EXT-X-MEDIA-SEQUENCE:1
#EXTINF:10,
fileSequence1.ts
#EXTINF:10,
fileSequence2.ts
#EXTINF:10,
fileSequence3.ts
#EXTINF:10,
fileSequence4.ts
#EXTINF:10,
fileSequence5.ts

```

```
#EXTM3U
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio",LANGUAGE="eng",NAME="English",AUTOSELECT=YES, \
DEFAULT=YES,URI="eng/prog_index.m3u8"
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio",LANGUAGE="fre",NAME="Français",AUTOSELECT=YES, \
DEFAULT=NO,URI="fre/prog_index.m3u8"
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="audio",LANGUAGE="sp",NAME="Espanol",AUTOSELECT=YES, \
DEFAULT=NO,URI="sp/prog_index.m3u8"

#EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=195023,CODECS="avc1.42e00a,mp4a.40.2",AUDIO="audio"
lo/prog_index.m3u8
#EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=591680,CODECS="avc1.42e01e,mp4a.40.2",AUDIO="audio"
hi/prog_index.m3u8
```

ExoPlayer  创建播放器的时候首先要根据媒体类型构建 Render，而 HlsRenderBuilder 在构建 Render 的时候首先要加载媒体清单（m3u8），从而决定 AudioRender 和 VideoRender 采用同一个数据源还是不同的数据源。

```java
private static final class AsyncRendererBuilder implements ManifestCallback<HlsPlaylist> {

  private final Context context;
  private final String userAgent;
  private final DemoPlayer player;
  private final ManifestFetcher<HlsPlaylist> playlistFetcher;

  private boolean canceled;

  public AsyncRendererBuilder(Context context, String userAgent, String url, DemoPlayer player) {
    this.context = context;
    this.userAgent = userAgent;
    this.player = player;
    HlsPlaylistParser parser = new HlsPlaylistParser();
	 // ManifestFetcher 负责从网络下载 m3u8 文件
    playlistFetcher = new ManifestFetcher<>(url, new DefaultUriDataSource(context, userAgent),
        parser);
  }

  public void init() {
  	  // 开始下载
    playlistFetcher.singleLoad(player.getMainHandler().getLooper(), this);
  }

  ...

  @Override
  public void onSingleManifest(HlsPlaylist manifest) {
    if (canceled) {
      return;
    }

    Handler mainHandler = player.getMainHandler();
    LoadControl loadControl = new DefaultLoadControl(new DefaultAllocator(BUFFER_SEGMENT_SIZE));
    DefaultBandwidthMeter bandwidthMeter = new DefaultBandwidthMeter();
    PtsTimestampAdjusterProvider timestampAdjusterProvider = new PtsTimestampAdjusterProvider();

    boolean haveSubtitles = false;
	  // 是否有独立的音频数据
    boolean haveAudios = false;
    if (manifest instanceof HlsMasterPlaylist) {
      HlsMasterPlaylist masterPlaylist = (HlsMasterPlaylist) manifest;
      haveSubtitles = !masterPlaylist.subtitles.isEmpty();
      haveAudios = !masterPlaylist.audios.isEmpty();
    }

    // Build the video/id3 renderers.
    // 负责从网络或文件中夹在数据
    DataSource dataSource = new DefaultUriDataSource(context, bandwidthMeter, userAgent);
    // Hls 有分块的概念,一个 chunk 加载完成之后换一个新的 chunk 继续加载，ChunkSource 负责创建新的 Chunk，并从
	  // 底层的 dataSource 获取数据
    HlsChunkSource chunkSource = new HlsChunkSource(true /* isMaster */, dataSource, manifest,
        DefaultHlsTrackSelector.newDefaultInstance(context), bandwidthMeter,
        timestampAdjusterProvider);
    // 负责为 Render 提供数据
    HlsSampleSource sampleSource = new HlsSampleSource(chunkSource, loadControl,
        MAIN_BUFFER_SEGMENTS * BUFFER_SEGMENT_SIZE, mainHandler, player, DemoPlayer.TYPE_VIDEO);
    MediaCodecVideoTrackRenderer videoRenderer = new MediaCodecVideoTrackRenderer(context,
        sampleSource, MediaCodecSelector.DEFAULT, MediaCodec.VIDEO_SCALING_MODE_SCALE_TO_FIT,
        5000, mainHandler, player, 50);
    MetadataTrackRenderer<List<Id3Frame>> id3Renderer = new MetadataTrackRenderer<>(
        sampleSource, new Id3Parser(), player, mainHandler.getLooper());

    // Build the audio renderer.
    MediaCodecAudioTrackRenderer audioRenderer;
    if (haveAudios) {
      // 如果有单独的音频数据，需要创建独立的数据源从网络加载数据
      DataSource audioDataSource = new DefaultUriDataSource(context, bandwidthMeter, userAgent);
      HlsChunkSource audioChunkSource = new HlsChunkSource(false /* isMaster */, audioDataSource,
          manifest, DefaultHlsTrackSelector.newAudioInstance(), bandwidthMeter,
          timestampAdjusterProvider);
      HlsSampleSource audioSampleSource = new HlsSampleSource(audioChunkSource, loadControl,
          AUDIO_BUFFER_SEGMENTS * BUFFER_SEGMENT_SIZE, mainHandler, player,
          DemoPlayer.TYPE_AUDIO);
      audioRenderer = new MediaCodecAudioTrackRenderer(
          new SampleSource[] {sampleSource, audioSampleSource}, MediaCodecSelector.DEFAULT, null,
          true, player.getMainHandler(), player, AudioCapabilities.getCapabilities(context),
          AudioManager.STREAM_MUSIC);
    } else {
 		//如果没有单独的音频数据，可以和视频 Render 使用相同的数据源
      audioRenderer = new MediaCodecAudioTrackRenderer(sampleSource,
          MediaCodecSelector.DEFAULT, null, true, player.getMainHandler(), player,
          AudioCapabilities.getCapabilities(context), AudioManager.STREAM_MUSIC);
    }

    // Build the text renderer.
    TrackRenderer textRenderer;
    if (haveSubtitles) {
      DataSource textDataSource = new DefaultUriDataSource(context, bandwidthMeter, userAgent);
      HlsChunkSource textChunkSource = new HlsChunkSource(false /* isMaster */, textDataSource,
          manifest, DefaultHlsTrackSelector.newSubtitleInstance(), bandwidthMeter,
          timestampAdjusterProvider);
      HlsSampleSource textSampleSource = new HlsSampleSource(textChunkSource, loadControl,
          TEXT_BUFFER_SEGMENTS * BUFFER_SEGMENT_SIZE, mainHandler, player, DemoPlayer.TYPE_TEXT);
      textRenderer = new TextTrackRenderer(textSampleSource, player, mainHandler.getLooper());
    } else {
      textRenderer = new Eia608TrackRenderer(sampleSource, player, mainHandler.getLooper());
    }

    TrackRenderer[] renderers = new TrackRenderer[DemoPlayer.RENDERER_COUNT];
    renderers[DemoPlayer.TYPE_VIDEO] = videoRenderer;
    renderers[DemoPlayer.TYPE_AUDIO] = audioRenderer;
    renderers[DemoPlayer.TYPE_METADATA] = id3Renderer;
    renderers[DemoPlayer.TYPE_TEXT] = textRenderer;
    player.onRenderers(renderers, bandwidthMeter);
  }

}

```

### 小结
这一步主要是从网络加载并解析 m3u8 文件，根据文件内容为 VideoRender 和 AudioRender 配置相同或不同的数据源。

## 拉流
这一步从 HlsSampleSource#maybeStartLoading()  开始，整个流程如下：


```
HlsSampleSource.maybeStartLoading() 
	HlsChunkSource.getChunkOperation //生成TsChunk
		loader.startLoading(loadable, this);//loadable 为 TsChunk，将其加入线程池
			TsChunk.load()
				HlsExtractorWrapper.read
					TsExtract.read
						DefaultExtractorInput.read
							DefaultUriDataSource.read
								DefaultHttpDataSource.read
```

TsExtract 中有个 tsPacketBuffer 缓存，它的大小是一个 Packet size，网络数据会被持续不断的放入这个缓存中，代码如下。

```java
TsChunk.load
@Override
public void load() throws IOException, InterruptedException {
  
  DataSpec loadDataSpec;
  boolean skipLoadedBytes;
  if (isEncrypted) {
    loadDataSpec = dataSpec;
    skipLoadedBytes = bytesLoaded != 0;
  } else {
    loadDataSpec = Util.getRemainderDataSpec(dataSpec, bytesLoaded);
    skipLoadedBytes = false;
  }

  try {
    ExtractorInput input = new DefaultExtractorInput(dataSource,
        loadDataSpec.absoluteStreamPosition, dataSource.open(loadDataSpec));
    if (skipLoadedBytes) {
      input.skipFully(bytesLoaded);
    }
    try {
      int result = Extractor.RESULT_CONTINUE;
	    // 循环读取知道块的尾部
      while (result == Extractor.RESULT_CONTINUE && !loadCanceled) {
        result = extractorWrapper.read(input);
      }
      long tsChunkEndTimeUs = extractorWrapper.getAdjustedEndTimeUs();
      if (tsChunkEndTimeUs != Long.MIN_VALUE) {
        adjustedEndTimeUs = tsChunkEndTimeUs;
      }
    } finally {
      bytesLoaded = (int) (input.getPosition() - dataSpec.absoluteStreamPosition);
    }
  } finally {
    Util.closeQuietly(dataSource);
  }
}

```
```java
TsExtractor.read
@Override
public int read(ExtractorInput input, PositionHolder seekPosition)
    throws IOException, InterruptedException {
  byte[] data = tsPacketBuffer.data;
  // 将 tsPacketBuffer 中的数据拷贝到头部
  if (BUFFER_SIZE - tsPacketBuffer.getPosition() < TS_PACKET_SIZE) {
    int bytesLeft = tsPacketBuffer.bytesLeft();
    if (bytesLeft > 0) {
      System.arraycopy(data, tsPacketBuffer.getPosition(), data, 0, bytesLeft);
    }
    tsPacketBuffer.reset(data, bytesLeft);
  }
  // 连续从 input 中读取数据，直到至少达到一个 packet size 的大小
  while (tsPacketBuffer.bytesLeft() < TS_PACKET_SIZE) {
    int limit = tsPacketBuffer.limit();
    int read = input.read(data, limit, BUFFER_SIZE - limit);
    if (read == C.RESULT_END_OF_INPUT) {
      return RESULT_END_OF_INPUT;
    }
    tsPacketBuffer.setLimit(limit + read);
  }

  ...
  
  boolean payloadUnitStartIndicator = tsScratch.readBit();
  tsScratch.skipBits(1); // transport_priority
  // 第 13 位是分组 ID 一个PID对应一种特定的PSI消息或者一个特定的PES。
  int pid = tsScratch.readBits(13);
  tsScratch.skipBits(2); // transport_scrambling_control
  boolean adaptationFieldExists = tsScratch.readBit();
  boolean payloadExists = tsScratch.readBit();

  ...
	// 不连续性检查等
  // Skip the adaptation field.

  // Read the payload.
  if (payloadExists) {
    TsPayloadReader payloadReader = tsPayloadReaders.get(pid);
    if (payloadReader != null) {
      if (discontinuityFound) {
        payloadReader.seek();
      }
      tsPacketBuffer.setLimit(endOfPacket);
      // 将数据交给特定的 payloadReader 处理
      payloadReader.consume(tsPacketBuffer, payloadUnitStartIndicator, output);
      Assertions.checkState(tsPacketBuffer.getPosition() <= endOfPacket);
      tsPacketBuffer.setLimit(limit);
    }
  }

  tsPacketBuffer.setPosition(endOfPacket);
  return RESULT_CONTINUE;
}

```

以上就是 Hls 拉流过程，它是按块加载的，每加载完一个 Chunk 会重新出发一次 maybeStartLoading()  来加载下一个块。
TsExtractor 中会每次读取一个 Packet size 大小的数据交给 payloadReader 处理。

## 解封装
解析 TS 流之前先了解一下 TS 流的文件格式：

[image:F82350DA-CD45-439B-9E45-108DD00FC66C-22562-0000CA015DE7563D/27100307_wUox.png]

>  ts层的内容是通过PID值来标识的，主要内容包括：PAT表、PMT表、音频流、视频流。解析ts流要先找到PAT表，只要找到PAT就可以找到PMT，然后就可以找到音视频流了。PAT表的PID值固定为0。PAT表和PMT表需要定期插入ts流，因为用户随时可能加入ts流，这个间隔比较小，通常每隔几个视频帧就要加入PAT和PMT。PAT和PMT表是必须的，还可以加入其它表如SDT（业务描述表）等，不过hls流只要有PAT和PMT就可以播放了。

要解析 TS 流首先要知道流中内容是什么编码方式，音频是 AAC 还是 DTS，视频是 H264 还是 H265，这些信息存储在 PMT 表中。而 PAT 表中存储着 program_number 及其对应的 PMT 的 PID。

所以 TS 流解析流程如下：
* 创建一个 PatReader 它的 pid 是 0。
* 解析 PAT 表，PMT pid 创建相应的 PmtReader。
* 解析 PMT 表，根据 stream_type 类型创建相应类型的内容 Reader。

详细内容格式请阅读[维基百科](https://zh.wikipedia.org/wiki/MPEG2-TS)

ExoPlayer 中具体代码如下：

```java
TsExtractor

public TsExtractor(PtsTimestampAdjuster ptsTimestampAdjuster, int workaroundFlags) {
   resetPayloadReaders();
}

private void resetPayloadReaders() {
  trackIds.clear();
  tsPayloadReaders.clear();
  // 创建 pid 为 0 的PatReader
  tsPayloadReaders.put(TS_PAT_PID, new PatReader());
  id3Reader = null;
  nextEmbeddedTrackId = BASE_EMBEDDED_TRACK_ID;
}

```
```java
TsExtractor#PatReader.consume
@Override
public void consume(ParsableByteArray data, boolean payloadUnitStartIndicator,
    ExtractorOutput output) {
  ...
  int programCount = (sectionLength - 9) / 4;
  for (int i = 0; i < programCount; i++) {
    sectionData.readBytes(patScratch, 4);
    int programNumber = patScratch.readBits(16);
    patScratch.skipBits(3); // reserved (3)
    if (programNumber == 0) {
      patScratch.skipBits(13); // network_PID (13)
    } else {
      // 如果存在 program，创建一个 PmtReader
      int pid = patScratch.readBits(13);
      tsPayloadReaders.put(pid, new PmtReader(pid));
    }
  }

}

```
```java
TsExtractor#PmtReader.consume

@Override
public void consume(ParsableByteArray data, boolean payloadUnitStartIndicator,
    ExtractorOutput output) {

  ...

  while (remainingEntriesLength > 0) {
    sectionData.readBytes(pmtScratch, 5);
    int streamType = pmtScratch.readBits(8);
    pmtScratch.skipBits(3); // reserved
    int elementaryPid = pmtScratch.readBits(13);
    pmtScratch.skipBits(4); // reserved
    int esInfoLength = pmtScratch.readBits(12); // ES_info_length
    if (streamType == 0x06) {
      // Read descriptors in PES packets containing private data.
      streamType = readPrivateDataStreamType(sectionData, esInfoLength);
    } else {
      sectionData.skipBytes(esInfoLength);
    }
    remainingEntriesLength -= esInfoLength + 5;
    int trackId = (workaroundFlags & WORKAROUND_HLS_MODE) != 0 ? streamType : elementaryPid;
    if (trackIds.get(trackId)) {
      continue;
    }
    ElementaryStreamReader pesPayloadReader;
    // 根据 streamType 创建相应类型的 Reader
    switch (streamType) {
      case TS_STREAM_TYPE_MPA:
        pesPayloadReader = new MpegAudioReader(output.track(trackId));
        break;
      case TS_STREAM_TYPE_MPA_LSF:
        pesPayloadReader = new MpegAudioReader(output.track(trackId));
        break;
      case TS_STREAM_TYPE_AAC:
        pesPayloadReader = (workaroundFlags & WORKAROUND_IGNORE_AAC_STREAM) != 0 ? null
            : new AdtsReader(output.track(trackId), new DummyTrackOutput());
        break;
      case TS_STREAM_TYPE_AC3:
        pesPayloadReader = new Ac3Reader(output.track(trackId), false);
        break;
      case TS_STREAM_TYPE_E_AC3:
        pesPayloadReader = new Ac3Reader(output.track(trackId), true);
        break;
      case TS_STREAM_TYPE_DTS:
      case TS_STREAM_TYPE_HDMV_DTS:
        pesPayloadReader = new DtsReader(output.track(trackId));
        break;
      case TS_STREAM_TYPE_H262:
        pesPayloadReader = new H262Reader(output.track(trackId));
        break;
      case TS_STREAM_TYPE_H264:
        pesPayloadReader = (workaroundFlags & WORKAROUND_IGNORE_H264_STREAM) != 0 ? null
            : new H264Reader(output.track(trackId),
                new SeiReader(output.track(nextEmbeddedTrackId++)),
                (workaroundFlags & WORKAROUND_ALLOW_NON_IDR_KEYFRAMES) != 0,
                (workaroundFlags & WORKAROUND_DETECT_ACCESS_UNITS) != 0);
        break;
      case TS_STREAM_TYPE_H265:
        pesPayloadReader = new H265Reader(output.track(trackId),
            new SeiReader(output.track(nextEmbeddedTrackId++)));
        break;
      case TS_STREAM_TYPE_ID3:
        if ((workaroundFlags & WORKAROUND_HLS_MODE) != 0) {
          pesPayloadReader = id3Reader;
        } else {
          pesPayloadReader = new Id3Reader(output.track(nextEmbeddedTrackId++));
        }
        break;
      default:
        pesPayloadReader = null;
        break;
    }

    if (pesPayloadReader != null) {
      trackIds.put(trackId, true);
      tsPayloadReaders.put(elementaryPid,
          new PesReader(pesPayloadReader, ptsTimestampAdjuster));
    }
  }
  if ((workaroundFlags & WORKAROUND_HLS_MODE) != 0) {
   if (!tracksEnded) {
     output.endTracks();
   }
  } else {
    tsPayloadReaders.remove(TS_PAT_PID);
    tsPayloadReaders.remove(pid);
    output.endTracks();
  }
  tracksEnded = true;
}

```

注意这里的 Reader 比如 H264Reader 被注入进 PesReader 里，之后才放入 tsPayloadReaders，这个由上面的图片可以看到音频或视频数据是放在 pes 里的所以要先做一次 pes 解析。

现在再回到 TsExtractor 的 read 方法，这里缓存完一个 packet size 的数据之后会根据当前数据的 pid 找到对应的 Reader 处理数据。比如 H264 数据就是由 PesReader 摘除掉 Header 之后交给 H264Reader 处理。

PesReader 就不看了，直接到 H264Reader，代码如下：

```java
@Override
public void consume(ParsableByteArray data) {
  while (data.bytesLeft() > 0) {
    int offset = data.getPosition();
    int limit = data.limit();
    byte[] dataArray = data.data;

    // Append the data to the buffer.
    totalBytesWritten += data.bytesLeft();
    output.sampleData(data, data.bytesLeft());

    // Scan the appended data, processing NAL units as they are encountered
    while (true) {
      int nalUnitOffset = NalUnitUtil.findNalUnit(dataArray, offset, limit, prefixFlags);

      if (nalUnitOffset == limit) {
        // We've scanned to the end of the data without finding the start of another NAL unit.
        nalUnitData(dataArray, offset, limit);
        return;
      }

      // We've seen the start of a NAL unit of the following type.
      int nalUnitType = NalUnitUtil.getNalUnitType(dataArray, nalUnitOffset);

      // This is the number of bytes from the current offset to the start of the next NAL unit.
      // It may be negative if the NAL unit started in the previously consumed data.
      int lengthToNalUnit = nalUnitOffset - offset;
      if (lengthToNalUnit > 0) {
        nalUnitData(dataArray, offset, nalUnitOffset);
      }
      int bytesWrittenPastPosition = limit - nalUnitOffset;
      long absolutePosition = totalBytesWritten - bytesWrittenPastPosition;
      // Indicate the end of the previous NAL unit. If the length to the start of the next unit
      // is negative then we wrote too many bytes to the NAL buffers. Discard the excess bytes
      // when notifying that the unit has ended.
      endNalUnit(absolutePosition, bytesWrittenPastPosition,
          lengthToNalUnit < 0 ? -lengthToNalUnit : 0, pesTimeUs);
      // Indicate the start of the next NAL unit.
      startNalUnit(absolutePosition, nalUnitType, pesTimeUs);
      // Continue scanning the data.
      offset = nalUnitOffset + 3;
    }
  }
}

```

这里以及后续的方法都是 NAL 相关处理，这里先跳过，以后有时间再分析。
音频流解析也不贴代码了。

## 解码
播放器初始化的时候创建了 MediaCodecAudioTrackRenderer 和 MediaCodecVideoTrackRenderer 做解码和渲染，从名字就可以看出来，它使用的是 MediaCodec 解码。

其中最关键的几个方法是：
* codec.dequeueInputBuffer  获取输入缓冲
* codec.queueInputBuffer  填充数据后放入队列
* codec.dequeueOutputBuffer 获取输出缓冲
* codec.releaseOutputBuffer 渲染后释放缓冲

MediaCodec 解码实现有需要的时候再分析，下面看一下 sample 数据从 SampleBuffer 到 MediaCodec  buffer 中的过程。

```java
SampleSourceTrackRenderer.doSomeWork
	MediaCodecTrackRenderer.doSomeWork
		MediaCodecTrackRenderer.feedInputBuffer
			SampleSourceTrackRenderer.readSource
				HlsSampleSource.readData
					HlsExtractorWrapper.getSample
						DefaultTrackOutput.getSample
							RollingSampleBuffer.readSample
								RollingSampleBuffer.readData
```

feedInputBuffer 这个方法中会创建一个 SampleHolder 一路传递下去，最后 readData 将 data 放入 SampleHolder 带回来。
解码音视频相同。

## 渲染
### 视频渲染
MediaCodec 如果创建的时候设置了 surface，在 native 层会创建一个 NativeWindow，如果 render 设置为 true 会在 releaseOutputBuffer 时会渲染一帧画面。

### 音频播放
 ExoPlayer 音频播放使用的是 AudioTrack，播放过程其实就是获取解码后的数据，然后写给 AudioTrack。

## 音视频同步
音视频同步 ExoPlayer 做的有点与众不同，它这里只用了一个线程做视频和音频的输出。
线程每隔 10ms 刷新一次，音频通过 AudioTrack 输出，写入数据后直接返回不阻塞，视频通过 MediaCodec 控制时间输出，也不阻塞。

```java
MediaCodecVideoTrackRenderer.processOutputBuffer
@Override
protected boolean processOutputBuffer(long positionUs, long elapsedRealtimeUs, MediaCodec codec,
    ByteBuffer buffer, MediaCodec.BufferInfo bufferInfo, int bufferIndex, boolean shouldSkip) {

  ...
  // positionUs 为音频的 pts，elapsedRealtimeUs 为本次刷新前记录的系统时间
  // Compute how many microseconds it is until the buffer's presentation time.
  // 开始刷新到现在已经过去的时间
  long elapsedSinceStartOfLoopUs = (SystemClock.elapsedRealtime() * 1000) - elapsedRealtimeUs;
  //视频 pts 减去 音频 pts 为 画面应该在上一帧声音多少时间内播放
  // 但是距离音频输出到现在已经过去一段时间，这段时间要减去即 - elapsedSinceStartOfLoopUs
  long earlyUs = bufferInfo.presentationTimeUs - positionUs - elapsedSinceStartOfLoopUs;

  // Compute the buffer's desired release time in nanoseconds.
  long systemTimeNs = System.nanoTime();
  long unadjustedFrameReleaseTimeNs = systemTimeNs + (earlyUs * 1000);

  // Apply a timestamp adjustment, if there is one.
  // 用一个工具计算一个更顺滑的 FrameReleaseNS，主要是用了两个方法
  // 1、多帧平均计算出 duration 使每帧过度更顺滑
  // 2、计算出一个离垂直同步更近的时间，使输出的画面能刚好被展示
  long adjustedReleaseTimeNs = frameReleaseTimeHelper.adjustReleaseTime(
      bufferInfo.presentationTimeUs, unadjustedFrameReleaseTimeNs);
  earlyUs = (adjustedReleaseTimeNs - systemTimeNs) / 1000;

  // 如果当前画面落后应该渲染时间 30 毫秒，丢帧
  if (shouldDropOutputBuffer(earlyUs, elapsedRealtimeUs)) {
    dropOutputBuffer(codec, bufferIndex);
    return true;
  }

  if (Util.SDK_INT >= 21) {
    // Let the underlying framework time the release.
    // Android 版本大于 21，MediaCodec 可以控制输出时间
    if (earlyUs < 50000) {
      renderOutputBufferV21(codec, bufferIndex, adjustedReleaseTimeNs);
      consecutiveDroppedFrameCount = 0;
      return true;
    }
  } else {
    // We need to time the release ourselves.
    if (earlyUs < 30000) {
      if (earlyUs > 11000) {
        // We're a little too early to render the frame. Sleep until the frame can be rendered.
        // Note: The 11ms threshold was chosen fairly arbitrarily.
        try {
          // Subtracting 10000 rather than 11000 ensures the sleep time will be at least 1ms.
          Thread.sleep((earlyUs - 10000) / 1000);
        } catch (InterruptedException e) {
          Thread.currentThread().interrupt();
        }
      }
      renderOutputBuffer(codec, bufferIndex);
      consecutiveDroppedFrameCount = 0;
      return true;
    }
  }


  return false;
}

```

```java
VideoFrameReleaseTimeHelper. adjustReleaseTime

public long adjustReleaseTime(long framePresentationTimeUs, long unadjustedReleaseTimeNs) {
  long framePresentationTimeNs = framePresentationTimeUs * 1000;

  // Until we know better, the adjustment will be a no-op.
  // 只有 haveSync 且 frameCount 大于一定数目之后才会重新计算，所以记录下来开始直接返回
  long adjustedFrameTimeNs = framePresentationTimeNs;
  long adjustedReleaseTimeNs = unadjustedReleaseTimeNs;

  if (haveSync) {
    // See if we've advanced to the next frame.
    if (framePresentationTimeUs != lastFramePresentationTimeUs) {
      frameCount++;
      adjustedLastFrameTimeNs = pendingAdjustedFrameTimeNs;
    }
    if (frameCount >= MIN_FRAMES_FOR_ADJUSTMENT) {
      // We're synced and have waited the required number of frames to apply an adjustment.
      // Calculate the average frame time across all the frames we've seen since the last sync.
      // This will typically give us a frame rate at a finer granularity than the frame times
      // themselves (which often only have millisecond granularity).
      // 多帧计算出一个 平均 durationNS
      long averageFrameDurationNs = (framePresentationTimeNs - syncFramePresentationTimeNs)
          / frameCount;
      // Project the adjusted frame time forward using the average.
      // 候选 frameTimeNS，frameTime 其实就是 presentionTime 在这里的的称呼
      // 上一帧的 frameTime + frameDuration 等于当前帧的 frameTime
      long candidateAdjustedFrameTimeNs = adjustedLastFrameTimeNs + averageFrameDurationNs;

      // 差距过大，跳过
      if (isDriftTooLarge(candidateAdjustedFrameTimeNs, unadjustedReleaseTimeNs)) {
        haveSync = false;
      } else {
        adjustedFrameTimeNs = candidateAdjustedFrameTimeNs;
        //计算出一个合适的的 releaseTime
        // 系统时间 + 当前 presentationTime（调整后的）- 第一帧的 presentationTime
        adjustedReleaseTimeNs = syncUnadjustedReleaseTimeNs + adjustedFrameTimeNs
            - syncFramePresentationTimeNs;
      }
    } else {
      // We're synced but haven't waited the required number of frames to apply an adjustment.
      // Check drift anyway.
      if (isDriftTooLarge(framePresentationTimeNs, unadjustedReleaseTimeNs)) {
        haveSync = false;
      }
    }
  }

  // If we need to sync, do so now.
  if (!haveSync) {
    syncFramePresentationTimeNs = framePresentationTimeNs;
    syncUnadjustedReleaseTimeNs = unadjustedReleaseTimeNs;
    frameCount = 0;
    haveSync = true;
    onSynced();
  }

  lastFramePresentationTimeUs = framePresentationTimeUs;
  pendingAdjustedFrameTimeNs = adjustedFrameTimeNs;

  if (vsyncSampler == null || vsyncSampler.sampledVsyncTimeNs == 0) {
    return adjustedReleaseTimeNs;
  }

  // Find the timestamp of the closest vsync. This is the vsync that we're targeting.
  // 计算出一个离垂直同步时间更近的时间
  long snappedTimeNs = closestVsync(adjustedReleaseTimeNs,
      vsyncSampler.sampledVsyncTimeNs, vsyncDurationNs);
  // Apply an offset so that we release before the target vsync, but after the previous one.
  return snappedTimeNs - vsyncOffsetNs;
}


```

小结：
* 音视频输出只用了一个线程，这个线程每隔 10ms 刷新一次。
* 音频写给 AudioTrack 后，直接返回不阻塞。
* 视频会根据多帧计算出一个平均 duration，使画面过度更平滑。
