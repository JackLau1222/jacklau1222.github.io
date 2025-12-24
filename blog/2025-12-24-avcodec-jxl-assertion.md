---
slug: ffmpeg-avcodec-jxl-assertion
title: FFmpeg是如何解码图像的-修复JXL解码assertion
authors: [jacklau]
tags: [ffmpeg, bug-fix, avcodec, c]
---

# FFmpeg的一大目标就是可以播放世界上所有媒体文件，本文以修复Jpeg XL一个bug为例，简要介绍FFmpeg是如何解码图像的

## 用户反馈使用mpv播放crop JXL图像的时候遇到assertion
```shell
[ffmpeg] Probing jpegxl_pipe score:98 size:20
[ffmpeg/demuxer] jpegxl_pipe: Before avformat_find_stream_info() pos: 0 bytes read:0 seeks:0 nb_streams:1
[ffmpeg/video] libjxl: BASIC_INFO event emitted
[ffmpeg/video] libjxl: COLOR_ENCODING event emitted
[ffmpeg/video] libjxl: FRAME event emitted
[ffmpeg/video] libjxl: NEED_IMAGE_OUT_BUFFER event emitted
[ffmpeg/video] libjxl: FULL_IMAGE event emitted
[ffmpeg/demuxer] jpegxl_pipe: stream 0: start_time: NOPTS duration: NOPTS
[ffmpeg/demuxer] jpegxl_pipe: format: start_time: NOPTS duration: NOPTS (estimate from bit rate) bitrate=0 kb/s
[ffmpeg/demuxer] jpegxl_pipe: After avformat_find_stream_info() pos: 20 bytes read:20 seeks:0 frames:1
● Image  --vid=1  (jpegxl 200x200)
[ffmpeg] detected 28 logical cores
[ffmpeg/video] libjxl: BASIC_INFO event emitted
[ffmpeg/video] libjxl: COLOR_ENCODING event emitted
[ffmpeg/video] libjxl: FRAME event emitted
[ffmpeg/video] libjxl: NEED_IMAGE_OUT_BUFFER event emitted
[ffmpeg/video] libjxl: FULL_IMAGE event emitted
[ffmpeg/video] libjxl: frame->private_ref: 0x558c555c8140
VO: [gpu-next] 200x200 gray
[ffmpeg/video] libjxl: SUCCESS event emitted
[ffmpeg/video] libjxl: frame->private_ref: (nil)
[ffmpeg] Assertion frame->private_ref || !(avctx->codec->capabilities & (1 << 1)) failed at src/libavcodec/decode.c:684
```

这是复现流程：
```shell
convert -size 200x200 xc:black image.png
cjxl -d 0 image.{png,jxl}

mkdir config
echo 'C vf toggle crop=in_w:in_w/2.4' > config/input.conf
mpv --config-dir=config image.jxl
<shift-c>
```
用户创建一张JXL图像，然后使用mpv播放，在播放过程中切换crop滤镜，就会触发assertion。

## 深入分析这个问题之前，我们先简单介绍一下FFmpeg解码的关键流程以及MPV的播放流程

### FFmpeg解码一帧图像的流程

省略一些初始化的流程，我们直接介绍最核心的两个API：

- avcodec_send_packet： 向编解码器发送一个packet，一个packet可以包含一帧或多帧数据
- avcodec_receive_frame ：从编解码器接收一帧图像数据

在这里需要先介绍一个概念，很多情况下一个packet里的数据不足以解码出完整的图像，所以会有一个内部缓冲区，直到收到足够多的packet或者一个空pkt表示EOF，才会输出

### MPV的播放流程

有一个关键的函数`lavc_process`包含了解码流程
```c
void lavc_process(struct mp_filter *f, struct lavc_state *state,
                  int (*send)(struct mp_filter *f, struct demux_packet *pkt),
                  int (*receive)(struct mp_filter *f, struct mp_frame *res))
{
    if (!mp_pin_in_needs_data(f->ppins[1]))
        return;

    struct mp_frame frame = {0};
    int ret_recv = receive(f, &frame);
    if (frame.type) {
        state->eof_returned = false;
        mp_pin_in_write(f->ppins[1], frame);
    } else if (ret_recv == AVERROR_EOF) {
        if (!state->eof_returned)
            mp_pin_in_write(f->ppins[1], MP_EOF_FRAME);
        state->eof_returned = true;
        state->packets_sent = false;
    } else if (ret_recv == AVERROR(EAGAIN)) {
        // Need to feed a packet.
        frame = mp_pin_out_read(f->ppins[0]);
        struct demux_packet *pkt = NULL;
        if (frame.type == MP_FRAME_PACKET) {
            pkt = frame.data;
        } else if (frame.type != MP_FRAME_EOF) {
            if (frame.type) {
                MP_ERR(f, "unexpected frame type\n");
                mp_frame_unref(&frame);
                mp_filter_internal_mark_failed(f);
            }
            return;
        } else if (!state->packets_sent) {
            // EOF only; just return it, without requiring send/receive to
            // pass it through properly.
            mp_pin_in_write(f->ppins[1], MP_EOF_FRAME);
            return;
        }
        int ret_send = send(f, pkt);
        if (ret_send == AVERROR(EAGAIN)) {
            // Should never happen, but can happen with broken decoders.
            MP_WARN(f, "could not consume packet\n");
            mp_pin_out_unread(f->ppins[0], frame);
            mp_filter_wakeup(f);
            return;
        }
        state->packets_sent = true;
        demux_packet_pool_push(f->packet_pool, pkt);
        mp_filter_internal_mark_progress(f);
    } else {
        // Decoding error, or hwdec fallback recovery. Just try again.
        mp_filter_internal_mark_progress(f);
    }
}
```
### 对于JXL图像解码，简单来说，这个函数的解码流程是：
1. avcodec_receive_frame尝试从解码器接收数据 由于还没发送packet，所以自然收不到frame，错误码是EAGAIN
2. avcodec_send_packet 发送第一个packet
3. avcodec_receive_frame 尝试从解码器接收数据，仍然为EAGAIN 这是因为需要我们再发送一个空packet表示EOF(没有更多数据了)
4. avcodec_send_packet 发送空packet
5. avcodec_receive_frame 成功接收到解码后的数据

这个正常的解码流程mpv是没有问题的，问题出在我们按下了`shift-c`切换滤镜，出现了assertion，让我们再看下具体日志：
```shell
[ffmpeg/video] libjxl: BASIC_INFO event emitted
[ffmpeg/video] libjxl: COLOR_ENCODING event emitted
[ffmpeg/video] libjxl: FRAME event emitted
[ffmpeg/video] libjxl: NEED_IMAGE_OUT_BUFFER event emitted
[ffmpeg/video] libjxl: FULL_IMAGE event emitted
[ffmpeg/video] libjxl: frame->private_ref: 0x558c555c8140
VO: [gpu-next] 200x200 gray
[ffmpeg/video] libjxl: SUCCESS event emitted
[ffmpeg/video] libjxl: frame->private_ref: (nil)
[ffmpeg] Assertion frame->private_ref || !(avctx->codec->capabilities & (1 << 1)) failed at src/libavcodec/decode.c:684
```

可以看到`SUCCESS event emitted`是按下`shift-c`切换滤镜后触发的，然后frame->private_ref就是null了，触发assertion

debug mpv代码得知，在切换滤镜后，mpv会flush codec buffer并seek一下，相当于重新解码一次图像，但是复用了原有解码上下文，就是这第二次解码图像出现了问题。

### 日志上显示SUCCESS event emitted，这个SUCCESS event对于JXL解码来说意味着解码完成，但是为什么这个事件触发在切换滤镜后，而不是在第一次解码完成后？

如果我们使用ffplay单独播放一张JXL图像可以看到
```shell
[libjxl @ 0x1368087d0] BASIC_INFO event emitted
[libjxl @ 0x1368087d0] COLOR_ENCODING event emitted
[libjxl @ 0x1368087d0] FRAME event emitted
[libjxl @ 0x1368087d0] NEED_IMAGE_OUT_BUFFER event emitted
[libjxl @ 0x1368087d0] FULL_IMAGE event emitted
```
正常情况下是不会看到success事件的，这是因为libjxldec代码在frame_complete后手动处理了“善后”工作
```c
                } else if (ctx->frame_complete) {
                    libjxl_finalize_frame(avctx, frame, ctx->frame);
                    ctx->jret = JXL_DEC_SUCCESS;
                    return 0;
                }
```
但是忘记reset了decoder状态，导致我们在切换滤镜后，重新发了一遍相同的packet，解码器自然认为已经解码完成了，就直接触发了SUCCESS事件，没有对其重新解码，而且mpv flush了codec buffer，所以frame里面没有数据，frame->private_ref自然也为空，就触发了assertion
```c
        case JXL_DEC_SUCCESS:
            av_log(avctx, AV_LOG_DEBUG, "SUCCESS event emitted\n");
            /*
             * this event will be fired when the zero-length EOF
             * packet is sent to the decoder by the client,
             * but it will also be fired when the next image of
             * an image2pipe sequence is loaded up
             */
            libjxl_finalize_frame(avctx, frame, ctx->frame);
            JxlDecoderReset(ctx->decoder);
            libjxl_init_jxl_decoder(avctx);
            return 0;
```

所以解决方案也很简单，就是把手动“善后”工作goto到JXL_DEC_SUCCESS事件处理中

修复patch已合并
https://code.ffmpeg.org/FFmpeg/FFmpeg/commit/13c91c97d12a28750f572c87cf13934456845df1
