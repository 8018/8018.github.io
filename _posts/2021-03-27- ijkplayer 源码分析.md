---
layout:     post
title:       ijkplayer 源码分析
date:       2021-03-27
author:     xflyme
header-img: img/post-bg-2021-03-27.jpg
catalog: true
tags:
    - 播放器
    - ijkplayer
    - 源码分析
---


ijkplayer 基于 FFMpeg 的 ffplay，本文将结合源码分析一下 ijkplayer 的基础实现，如果不了解 FFMpeg 的，建议先看一下 FFMpeg。

播放器基本流程如下：

![播放流程](/img/ijkplayer-1.png)

## 播放器创建
```c
IJKMediaPlayer.initPlayer
	jikplayer_jni.IjkMediaPlayer_native_setup
		ijkplayer_android.ijkmp_android_create
			ijkplayer.ijkmp_create
				ff_ffplay.ffp_create
					ijkmeta.ijkmeta_create		
```

以上是播放器创建的一个流程，主要是在 c 层创建了一个 FFPlayer。

## 打开流以及解复用流程
```c
IjkMediaPlayer.prepareAsync
	ijkplayer_jni._prepareAsync
		ijkplayer.ijkmp_prepare_async
			ijkplayer.ijkmp_prepare_async_l
				ff_ffplay.ffp_prepare_async_l
					ff_ffplay.stream_open
						ff_ffplay.read_thread
```

以上流程中重点在 stream_open 和 read_thread，stream_open 进行一些初始化，read_thread 打开流、解复用以及创建三个解码线程。

### stream_open

```c
static VideoState *stream_open(FFPlayer *ffp, const char *filename, AVInputFormat *iformat)
{
    assert(!ffp->is);
    VideoState *is;
    // 创建 VideoState
    is = av_mallocz(sizeof(VideoState));
    ...
#if defined(__ANDROID__)
	  // 创建 soundtouch，用于变速变调 
    if (ffp->soundtouch_enable) {
        is->handle = ijk_soundtouch_create();
    }
#endif

    /* start video display */
    // 创建 frame 队列，用于存储解码后数据
    if (frame_queue_init(&is->pictq, &is->videoq, ffp->pictq_size, 1) < 0)
        goto fail;
    if (frame_queue_init(&is->subpq, &is->subtitleq, SUBPICTURE_QUEUE_SIZE, 0) < 0)
        goto fail;
    if (frame_queue_init(&is->sampq, &is->audioq, SAMPLE_QUEUE_SIZE, 1) < 0)
        goto fail;
	  // 创建 packet 队列，用于存储解复用后数据
    if (packet_queue_init(&is->videoq) < 0 ||
        packet_queue_init(&is->audioq) < 0 ||
        packet_queue_init(&is->subtitleq) < 0)
        goto fail;
	  ...
	  // 初始化 clock ，用于音视频同步
    init_clock(&is->vidclk, &is->videoq.serial);
    init_clock(&is->audclk, &is->audioq.serial);
    init_clock(&is->extclk, &is->extclk.serial);

    ...
	  // 创建视频渲染线程
    is->video_refresh_tid = SDL_CreateThreadEx(&is->_video_refresh_tid, video_refresh_thread, ffp, "ff_vout");
    if (!is->video_refresh_tid) {
        av_freep(&ffp->is);
        return NULL;
    }

	  //创建 read 线程，解复用以及三个解码线程在其中创建
    is->initialized_decoder = 0;
    is->read_tid = SDL_CreateThreadEx(&is->_read_tid, read_thread, ffp, "ff_read");
    ...

    return is;
fail:
    is->initialized_decoder = 1;
    is->abort_request = true;
    if (is->video_refresh_tid)
        SDL_WaitThread(is->video_refresh_tid, NULL);
    stream_close(ffp);
    return NULL;
}

```


### read_thread

```c
read_thread(void *arg)
{
    FFPlayer *ffp = arg;
    VideoState *is = ffp->is;
    AVFormatContext *ic = NULL;
    int err, i, ret __unused;
    int st_index[AVMEDIA_TYPE_NB];
    AVPacket pkt1, *pkt = &pkt1;
    int64_t stream_start_time;
    int completed = 0;
    int pkt_in_play_range = 0;
    AVDictionaryEntry *t;
    SDL_mutex *wait_mutex = SDL_CreateMutex();
    int scan_all_pmts_set = 0;
    int64_t pkt_ts;
    int last_error = 0;
    int64_t prev_io_tick_counter = 0;
    int64_t io_tick_counter = 0;
    int init_ijkmeta = 0;

    ...
	  //创建 AVFormatContext 对象，FFMpeg 视频播放的第一步
    ic = avformat_alloc_context();
    ...
	  //打开视频流
    err = avformat_open_input(&ic, is->filename, is->iformat, &ffp->format_opts);
    ...
    if (!ffp->video_disable)
        st_index[AVMEDIA_TYPE_VIDEO] =
            av_find_best_stream(ic, AVMEDIA_TYPE_VIDEO,
                                st_index[AVMEDIA_TYPE_VIDEO], -1, NULL, 0);
    if (!ffp->audio_disable)
        st_index[AVMEDIA_TYPE_AUDIO] =
            av_find_best_stream(ic, AVMEDIA_TYPE_AUDIO,
                                st_index[AVMEDIA_TYPE_AUDIO],
                                st_index[AVMEDIA_TYPE_VIDEO],
                                NULL, 0);
    if (!ffp->video_disable && !ffp->subtitle_disable)
        st_index[AVMEDIA_TYPE_SUBTITLE] =
            av_find_best_stream(ic, AVMEDIA_TYPE_SUBTITLE,
                                st_index[AVMEDIA_TYPE_SUBTITLE],
                                (st_index[AVMEDIA_TYPE_AUDIO] >= 0 ?
                                 st_index[AVMEDIA_TYPE_AUDIO] :
                                 st_index[AVMEDIA_TYPE_VIDEO]),
                                NULL, 0);

    is->show_mode = ffp->show_mode;
#ifdef FFP_MERGE // bbc: dunno if we need this
    if (st_index[AVMEDIA_TYPE_VIDEO] >= 0) {
        AVStream *st = ic->streams[st_index[AVMEDIA_TYPE_VIDEO]];
        AVCodecParameters *codecpar = st->codecpar;
        AVRational sar = av_guess_sample_aspect_ratio(ic, st, NULL);
        if (codecpar->width)
            set_default_window_size(codecpar->width, codecpar->height, sar);
    }
#endif

    /* open the streams */
	  // stream_component_open，根据流的类型创建解码器以及解码线程 稍后分析
    if (st_index[AVMEDIA_TYPE_AUDIO] >= 0) {
        stream_component_open(ffp, st_index[AVMEDIA_TYPE_AUDIO]);
    } else {
        ffp->av_sync_type = AV_SYNC_VIDEO_MASTER;
        is->av_sync_type  = ffp->av_sync_type;
    }

    ret = -1;
    if (st_index[AVMEDIA_TYPE_VIDEO] >= 0) {
        ret = stream_component_open(ffp, st_index[AVMEDIA_TYPE_VIDEO]);
    }
    if (is->show_mode == SHOW_MODE_NONE)
        is->show_mode = ret >= 0 ? SHOW_MODE_VIDEO : SHOW_MODE_RDFT;

    if (st_index[AVMEDIA_TYPE_SUBTITLE] >= 0) {
        stream_component_open(ffp, st_index[AVMEDIA_TYPE_SUBTITLE]);
    }
    ...
	  // 循环从流中读取数据
    for (;;) {
        ...
		  // 从流中读取一个 packet
        ret = av_read_frame(ic, pkt);
        ...
		  // 根据 packet 类型放入相应的队列
        if (pkt->flags & AV_PKT_FLAG_DISCONTINUITY) {
            if (is->audio_stream >= 0) {
                packet_queue_put(&is->audioq, &flush_pkt);
            }
            if (is->subtitle_stream >= 0) {
                packet_queue_put(&is->subtitleq, &flush_pkt);
            }
            if (is->video_stream >= 0) {
                packet_queue_put(&is->videoq, &flush_pkt);
            }
        }

       ...
    }

    ret = 0;
 	  ...
    return 0;
}

```

### stream_component_open

```c
static int stream_component_open(FFPlayer *ffp, int stream_index)
{
    VideoState *is = ffp->is;
    AVFormatContext *ic = is->ic;
    AVCodecContext *avctx;
    AVCodec *codec = NULL;
    const char *forced_codec_name = NULL;
    AVDictionary *opts = NULL;
    AVDictionaryEntry *t = NULL;
    int sample_rate, nb_channels;
    int64_t channel_layout;
    int ret = 0;
    int stream_lowres = ffp->lowres;

    ..
	  // 创建解码器
    codec = avcodec_find_decoder(avctx->codec_id);

    ...
	  // 根据流的类型创建输出设备以及解码线程
    switch (avctx->codec_type) {
    case AVMEDIA_TYPE_AUDIO:
        
        ...

        /* prepare audio output */
        if ((ret = audio_open(ffp, channel_layout, nb_channels, sample_rate, &is->audio_tgt)) < 0)
            goto fail;
        ...
        decoder_init(&is->auddec, avctx, &is->audioq, is->continue_read_thread);
        if ((is->ic->iformat->flags & (AVFMT_NOBINSEARCH | AVFMT_NOGENSEARCH | AVFMT_NO_BYTE_SEEK)) && !is->ic->iformat->read_seek) {
            is->auddec.start_pts = is->audio_st->start_time;
            is->auddec.start_pts_tb = is->audio_st->time_base;
        }
		  // 创建解码线程
        if ((ret = decoder_start(&is->auddec, audio_thread, ffp, "ff_audio_dec")) < 0)
            goto out;
        SDL_AoutPauseAudio(ffp->aout, 0);
        break;
    case AVMEDIA_TYPE_VIDEO:
        
		  ...
		  // 创建渲染管线
        if (ffp->async_init_decoder) {
            while (!is->initialized_decoder) {
                SDL_Delay(5);
            }
            if (ffp->node_vdec) {
                is->viddec.avctx = avctx;
                ret = ffpipeline_config_video_decoder(ffp->pipeline, ffp);
            }
            if (ret || !ffp->node_vdec) {
                decoder_init(&is->viddec, avctx, &is->videoq, is->continue_read_thread);
                ffp->node_vdec = ffpipeline_open_video_decoder(ffp->pipeline, ffp);
                if (!ffp->node_vdec)
                    goto fail;
            }
        } else {
            decoder_init(&is->viddec, avctx, &is->videoq, is->continue_read_thread);
            ffp->node_vdec = ffpipeline_open_video_decoder(ffp->pipeline, ffp);
            if (!ffp->node_vdec)
                goto fail;
        }
  		  // 创建视频解码线程
        if ((ret = decoder_start(&is->viddec, video_thread, ffp, "ff_video_dec")) < 0)
            goto out;
		  ...

        break;
    case AVMEDIA_TYPE_SUBTITLE:
        ...
        break;
    default:
        break;
    }
    goto out;

 	...
    return ret;
}

```

### 小结
* stream_open 进行一些参数、队列的初始化，打开流并打开 read_thread 线程
* read_thread 进行解复用，将解出 AVPacket 放入相应的队列，并根据流的数据不同打开 stream_component_open
* stream_component_open 根据流的不同创建相应的解码器并打开相应的解码线程

## 音频解码 & 渲染
### 音频输出工具创建
Android 音频输出可以使用 AudioTrack 也可以使用 OpenSl，音频输出根据条件创建，具体流程如下：

```c
ijkplayer.ijkmp_prepare_async_l
	ff_ffplay.ffp_prepare_async_l
		ff_pipeline.ffpipeline_open_audio_output
			ffpipeline_android.func_open_audio_output
```

在音频解码、渲染这个流程中，解码器和渲染器是生产者消费者的关系，解码器负责解码并将解码结果放入 Frame 队列，渲染器负责将解码后的结果输出。

### 解码

```c
static int audio_thread(void *arg)
{
    FFPlayer *ffp = arg;
    VideoState *is = ffp->is;
    AVFrame *frame = av_frame_alloc();
    Frame *af;
	  ...
    // 循环解码
    do {
        ffp_audio_statistic_l(ffp);
        if ((got_frame = decoder_decode_frame(ffp, &is->auddec, frame, NULL)) < 0)
            goto the_end;

        if (got_frame) {
                tb = (AVRational){1, frame->sample_rate};
                
				  // 跳过 seek 以及 avfilter 等代码
				  // 从 FrameQueue 中获取一个可写的 Frame
                if (!(af = frame_queue_peek_writable(&is->sampq)))
                    goto the_end;

                af->pts = (frame->pts == AV_NOPTS_VALUE) ? NAN : frame->pts * av_q2d(tb);
                af->pos = frame->pkt_pos;
                af->serial = is->auddec.pkt_serial;
                af->duration = av_q2d((AVRational){frame->nb_samples, frame->sample_rate});

                av_frame_move_ref(af->frame, frame);
				  // 将 Frame 放入队列中
                frame_queue_push(&is->sampq);

				  ...
    } while (ret >= 0 || ret == AVERROR(EAGAIN) || ret == AVERROR_EOF);
	  // 释放 frame	
    av_frame_free(&frame);
    return ret;
}

```

audio_thread 主要是将 AVPacket 解码成 AVFrame，然后将其转换成 Frame 放入队列中等待消费者消费。

### 音频输出（以 OpenSl 为例）

输出调用链

```c
ff_ffplay.stream_component_open
	ff_ffplay.audio_open
		ijksdl_aout.SDL_AoutOpenAudio
			ijksdl_aout_android_opensles.aout_open_audio
				ijksdl_aout_android_opensles.aout_thread
					ijksdl_aout_android_opensles.aout_thread_n
						ff_ffplay.sdl_audio_callback
							ff_ffplay.audio_decode_frame
```

其中 aout_open_audio 中进行 OpenSLES 的初始化，aout_thread_n 开启一个循环等待 OpenSLES 回调。回调时会从 Frame 队列中取出一个 Frame 供 OpenSLES 输出。

如果有倍速播放等，ff_ffplay.audio_decode_frame 还会使用 soundtouch 进行转换以调整音频播放时长。
具体代码就不再一一列出了。

## 视频解码 & 渲染
视频解码有两种方式：硬解、软解。ijkplayer 中可以根据配置选择解码器。

```c
ffpipeline_android.c
static IJKFF_Pipenode *func_open_video_decoder(IJKFF_Pipeline *pipeline, FFPlayer *ffp)
{
    IJKFF_Pipeline_Opaque *opaque = pipeline->opaque;
    IJKFF_Pipenode        *node = NULL;

    if (ffp->mediacodec_all_videos || ffp->mediacodec_avc || ffp->mediacodec_hevc || ffp->mediacodec_mpeg2)
        node = ffpipenode_create_video_decoder_from_android_mediacodec(ffp, pipeline, opaque->weak_vout);
    if (!node) {
        node = ffpipenode_create_video_decoder_from_ffplay(ffp);
    }

    return node;
}
```


### 视频解码
根据解码方式的不同也会调用不同的方法，具体方法软解是 ff_ffplay 中的 ffplay_video_thread，硬解是 ffpipenode_android_mediacodec_vdec 中的 func_run_sync
。

#### 软解
```c
static int ffplay_video_thread(void *arg)
{
    FFPlayer *ffp = arg;
    VideoState *is = ffp->is;
    AVFrame *frame = av_frame_alloc();
    double pts;
    double duration;
    
    ...
    // 循环解码
    for (;;) {
		  // 解码一帧
        ret = get_video_frame(ffp, frame);
        

			// 图片入队
          ret = queue_picture(ffp, frame, pts, duration, frame->pkt_pos, is->viddec.pkt_serial);
          av_frame_unref(frame);

        }
		...
    return 0;
}

```

```c
static int get_video_frame(FFPlayer *ffp, AVFrame *frame)
{
    VideoState *is = ffp->is;
    int got_picture;

	  // 解码
    if ((got_picture = decoder_decode_frame(ffp, &is->viddec, frame, NULL)) < 0)
        return -1;
		...
    return got_picture;
}

```

```c
static int decoder_decode_frame(FFPlayer *ffp, Decoder *d, AVFrame *frame, AVSubtitle *sub) {
    int ret = AVERROR(EAGAIN);

	  // 结果不是 error 的情况下，循环获取 frame
    for (;;) {
        AVPacket pkt;

        if (d->queue->serial == d->pkt_serial) {
            do {
                switch (d->avctx->codec_type) {
                    case AVMEDIA_TYPE_VIDEO:
                        ret = avcodec_receive_frame(d->avctx, frame);
                        ...
                        break;
                    case AVMEDIA_TYPE_AUDIO:
                        ret = avcodec_receive_frame(d->avctx, frame);
                        ...
                        break;
                    default:
                        break;
                }
                ...
            } while (ret != AVERROR(EAGAIN));
        }

       ...
        if (pkt.data == flush_pkt.data) {
           ...
        } else {
            if (d->avctx->codec_type == AVMEDIA_TYPE_SUBTITLE) {
               ...
            } else {
				 // 送入一个 packet
                if (avcodec_send_packet(d->avctx, &pkt) == AVERROR(EAGAIN)) {
                    av_log(d->avctx, AV_LOG_ERROR, "Receive_frame and send_packet both returned EAGAIN, which is an API violation.\n");
                    d->packet_pending = 1;
                    av_packet_move_ref(&d->pkt, &pkt);
                }
            }
            av_packet_unref(&pkt);
        }
    }
}
```

decoder_decode_frame 中会根据送入的 Packet 类型输出相应类型的 Frame，因为 I、B、P 不同类型的帧的存在，有时候输入之后没有输出，有时候输入一个 packet 会有多帧输出，所以这里写了一个循环持续的获取数据。

#### 硬解
MediaCodec 主要流程如下：
* MediaCodec.queueInputBuffer 给 MediaCodec 喂数据
* MediaCodec.dequeueOutputBuffer 从 MediaCodec 中获取解码数据
* 将解码后的 Frame 存入 FrameQueue

MediaCodec 具体的代码以及使用方法以后有时间再分析。

## 音视频同步 & 倍速播放

### 音视频同步

音视频同步一般以音频作为主时钟，视频向音频看齐，本文也将只分析这种情况。以视频作为主时钟或以外部时钟作为主时钟的以后有时间再分析。

```c
typedef struct VideoState {
    ...

    Clock audclk;
    Clock vidclk;
    Clock extclk;

    ...
} VideoState;

typedef struct Clock {
    double pts;           /* clock base */
    double pts_drift;     /* clock base minus time at which we updated the clock */
    double last_updated;
    double speed;
    int serial;           /* clock is based on a packet with this serial */
    int paused;
    int *queue_serial;    /* pointer to the current packet queue serial, used for obsolete clock detection */
} Clock;

```


VideoState 中定义了三个 Clock，这三个 Clock 在 stream_open 时被初始化。音频播放的时候会在 sdl_audio_callback 中修改 audclk。

```c
ff_ffplay.c
sdl_audio_callback(void *opaque, Uint8 *stream, int len)
{
    ...

    if (!isnan(is->audio_clock)) {
        set_clock_at(&is->audclk, is->audio_clock - (double)(is->audio_write_buf_size) / is->audio_tgt.bytes_per_sec - SDL_AoutGetLatencySeconds(ffp->aout), is->audio_clock_serial, ffp->audio_callback_time / 1000000.0);
        sync_clock_to_slave(&is->extclk, &is->audclk);
    }

    ...
}

```
 
```c
ff_ffplay.c
static void set_clock_at(Clock *c, double pts, int serial, double time)
{
    c->pts = pts;
    c->last_updated = time;
    c->pts_drift = c->pts - time;
    c->serial = serial;
}
```

其中 last_updated 时当前相对时间，pts_drift 是 pts 和当前相对时间的差值。

video_refresh 中会根据视频当前帧的 duration、视频 clock、音频 clock 计算出一个 remaining_time，然后通过这个 remaining_time 来控制下一帧的开始播放时间。

```c
remaining_time)
{
    
    double time;

    if (is->video_st) {
retry:
        if (frame_queue_nb_remaining(&is->pictq) == 0) {
            // nothing to do, no picture to display in the queue
        } else {
            double last_duration, duration, delay;
            Frame *vp, *lastvp;

            /* dequeue the picture */
            lastvp = frame_queue_peek_last(&is->pictq);
            vp = frame_queue_peek(&is->pictq);

            if (vp->serial != is->videoq.serial) {
                frame_queue_next(&is->pictq);
                goto retry;
            }

            if (lastvp->serial != vp->serial)
                is->frame_timer = av_gettime_relative() / 1000000.0;

            if (is->paused)
                goto display;

            /* compute nominal last_duration */
			  - // double duration = nextvp->pts - vp->pts;
            last_duration = vp_duration(is, lastvp, vp);
            delay = compute_target_delay(ffp, last_duration, is);

            time= av_gettime_relative()/1000000.0;
            if (isnan(is->frame_timer) || time < is->frame_timer)
                is->frame_timer = time;
            if (time < is->frame_timer + delay) {
                *remaining_time = FFMIN(is->frame_timer + delay - time, *remaining_time);
                goto display;
            }

            is->frame_timer += delay;
            if (delay > 0 && time - is->frame_timer > AV_SYNC_THRESHOLD_MAX)
                is->frame_timer = time;

            SDL_LockMutex(is->pictq.mutex);
            if (!isnan(vp->pts))
                update_video_pts(is, vp->pts, vp->pos, vp->serial);
            SDL_UnlockMutex(is->pictq.mutex);

			  // 丢帧
            if (frame_queue_nb_remaining(&is->pictq) > 1) {
                Frame *nextvp = frame_queue_peek_next(&is->pictq);
                duration = vp_duration(is, vp, nextvp);
				  //time > is->frame_timer + duration 上一次刷新时间加上 duration 仍然小于相对时间，说明视频超前,丢帧 retry 下一帧
                if(!is->step && (ffp->framedrop > 0 || (ffp->framedrop && get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER)) 
						&& time > is->frame_timer + duration) {
                    frame_queue_next(&is->pictq);
                    goto retry;
                }
            }

            ...
        }
display:
        /* display picture */
        if (!ffp->display_disable && is->force_refresh && is->show_mode == SHOW_MODE_VIDEO && is->pictq.rindex_shown)
            video_display2(ffp);
    }
    ...
    }
}

```

```c
static double compute_target_delay(FFPlayer *ffp, double delay, VideoState *is)
{
    double sync_threshold, diff = 0;

    /* update delay to follow master synchronisation source */
    if (get_master_sync_type(is) != AV_SYNC_VIDEO_MASTER) {
        /* if video is slave, we try to correct big delays by
           duplicating or deleting a frame */
        diff = get_clock(&is->vidclk) - get_master_clock(is);

        /* skip or repeat frame. We take into account the
           delay to compute the threshold. I still don't know
           if it is the best guess */
        sync_threshold = FFMAX(AV_SYNC_THRESHOLD_MIN, FFMIN(AV_SYNC_THRESHOLD_MAX, delay));
        /* -- by bbcallen: replace is->max_frame_duration with AV_NOSYNC_THRESHOLD */
        if (!isnan(diff) && fabs(diff) < AV_NOSYNC_THRESHOLD) {
			  // diff 小于 0 说明视频慢于音频，需要减少视频显示时间来完成同步，但 delay 不能小于 0
            if (diff <= -sync_threshold)
                delay = FFMAX(0, delay + diff);
			  // delay 当前帧需要显示的时间，AV_SYNC_FRAMEDUP_THRESHOLD 是一个阀值
			  // 也就是说如果当前帧显示时间过长，则不复制帧同步
            else if (diff >= sync_threshold && delay > AV_SYNC_FRAMEDUP_THRESHOLD)
                delay = delay + diff;
			  // diff 大于阀值，也就是说视频快于音频，当前帧显示时间加倍以缩小音视频差距
            else if (diff >= sync_threshold)
                delay = 2 * delay;
        }
    }

   ...

    return delay;
}

```

小结
* 音视频同步以音频为主时钟
* 视频通过 remaining_time 控制每帧显示时间
* remaining_time 计算公式为 frame_timer + delay - time，其中 frame_timer 为上一帧开始展示时间，delay 为结合当前帧显示时间以及视频时钟同主时钟差距总和计算出来的一个值。

### 倍速播放
音视频同步视频这块只通过 remaining_time 控制每帧显示时间，speed、playback_rate 相关的参数在视频播放这块完全没有用到，那么怎么进行的倍速播放？
答案是*视频同步音频，音频播放速度加快了，视频播放速度也会加快，这样就可能导致某些帧被跳过以加快视频播放速度。速度变慢也类似，音频变慢，那么视频快于音频，这是会通过增加 delay 控制播放速度。*

