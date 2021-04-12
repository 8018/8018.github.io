---
layout:     post
title:      ExoPlayer 源码分析 二 类图 & 名词解释
date:       2021-04-02
author:     xflyme
header-img: img/post-bg-2021-04-02.jpeg
catalog: true
tags:
    - 音视频
    - ExoPlayer
    - 源码分析
---


本文基于 ExoPlayer 2.13.2 版。

## 线程相关

![图一](/img/exoplayer-1.png)


##### LoadTask 
LoadTask 实现了 Runnable 接口，而且同时继承了 Handler ，它有三个作用：
* 通过 Loadable 加载数据—— run 方法中调用 Loadable 的 load 方法。
* load 完成之后通过 handler 通知 callback。
* 加载过程中如有异常发生，通知 callback。

##### Loadable 
真正执行加载工作的类，它的两个实现后续再分析。

##### Loader
Loader 中有个线程池，当有工作要做的时候它会创建一个 LoadTask 丢进线程池中之行。

## 配置文件

### HLS

![图二](/img/exoplayer-2.png)



##### HlsMediaPlaylist 
HlsMediaPlaylist 可以简单理解为一路可播放的流，其中的 Segment 表示这一路流被切成小的分段，播放时 Segment 按顺序加载。

##### HlsMasterPlayList
Alternate Media 支持多种码率或多种语言的流，这时候就需要多级目录，HlsMasterPlayList 可以看作是一级目录，其中的每个 Variant 解析出来又是 HlsMediaPlaylist。

### DASH

![图三](/img/exoplayer-3.png)


这个类图各个组件很好的对应了 MPD 文件：

![图四](/img/exoplayer-4.png)


##### MediaPresentationDescription（MPD）
MPD 是一种分层数据文件，用 xml 表示，一个 MPD 描述了视频的所有信息，一个 MPD 包含一个或多个 Period。

##### Period
每个 Period 代表一个时间段，同一个时间段内，可用的媒体内容以及码率不会变更。如果是直播可能需要周期的去服务器更新 MPD 文件，服务器可能会移除旧的已经过时的Period,或是添加新的Period。新的Period中可能会添加新的可用码率或去掉上一个Period中存在的某些码率。

##### AdaptationSet 
一个 Period 由一个或多个 AdaptationSet 组成，AdaptationSet 由一组可供切换不同码率的流组成。

##### Representation
每个 Adaptationset 包含了一个或者多个Representations.

##### Segment
每个Representation由一个或者多个segment组成 ;

## DataSource
![图五](/img/exoplayer-5.png)


##### DataSource 
DataSource 负责提供媒体数据，DefaultHttpDataSource 等就是它不同形式的实现，这个比较好理解就不一一解释了。

## Chunk
![图六](/img/exoplayer-6.png)


##### Chunk 
Chunk 实现了 Loadable 接口，它可以放入线程中执行，用于加载数据。Hls 和 DASH 都是分段加载的，一个 Chunk 的子类负责加载一个 Segment 的数据（这里不确定 Chunk 和 Segment 的关系是否一对一，有时间再详细分析）。

##### TsChunk 
用于加载 Hls 中的 MPEG2TS 块。

##### MediaPlaylistChunk
处理 Hls 配置文件的 chunk。

##### SingleSampleMediaChunk
VTT 等字幕文件是不需要分块加载的，可以一次性加载完，SingleSampleMediaChunk 用于处理这种情况。

##### ContainerMediaChunk
DASH 和 SmoothStreaming 都是用它加载和解析分段的多媒体数据。

## Extractor
![图七](/img/exoplayer-7.png)


##### Extractor 
顾名思义就是内容提取器，从原始数据流中提取出音频、视频、字幕等内容。具体提取方式由子类实现。比如 TsExtractor  构造了一系列 PayLoadReader，通过他们实现，而 Mp4Extractor 则是自己实现。

##### ExtractorInput
提供数据供 Extractor 消费，它的默认实现是 DefaultExtractorInput 数据源是 DataSource。

##### ExtractorOutput
接收 Extractor 提取的流级数据。

##### TrackOutput 
接收由 Extractor 提取的数据，它的默认实现是 DefaultTrackOut，它内部通过 RollingSampleBuffer 存储数据。

##### RollingSampleBuffer
样本数据和相应样本信息的滚动缓冲区，存储数据的容器是阻塞的双端队列，RollingSampleBuffer 将它封装成滚动的形式。

## ChunkSource
![图八](/img/exoplayer-8.png)

##### ChunkSource
用于构建并提供 Chunk 对象，上面说过 Chunk 是一种 Loadable ，它负责从网络、file 等加载数据。HlsChunkSource 并未实现 ChunkSource 接口。

## SampleSource

![图九](/img/exoplayer-9.png)


##### SampleHolder
Extractor 提取后的数据存储在 RollingSampleBuffer 中，这里的数据比如一个 nal unit 就称为 Sample。SampleHolder 里有一个 buffer 用于暂存 Sample 数据供解码器使用。

##### SampleSource 
为解码器提供 Sample 数据。

##### SampleSourceReader 

从 RollingSampleBuffer 读取 Sample 数据。

##### Render

![图十](/img/exoplayer-10.png)


Renderer 的子类并不只负责渲染，而是负责读取 Sample、解码、渲染这三个行为，Renderer 通过继承将公共的行为抽象出来。比如 SampleSourceTrackRenderer 负责从 RollingSampleBuffer 中读取数据，MediaCodecTrackRenderer 负责初始化 MediaCodec 并将数据写入 MediaCodec 的 inputBuffer 中。而
MediaCodecAudioTrackRenderer 中则创建了 AudioTrack 负责音频输出。

## Hls 相关类图
![图十一](/img/exoplayer-11.png)
























































