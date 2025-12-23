---
slug: ffmpeg-avformat-probe-amr
title: FFmpeg是如何探测文件格式的-增强AMR探测
authors: [jacklau]
tags: [ffmpeg, bug-fix, avformat, c]
---

## FFmpeg可能是世界上能识别最多媒体文件格式的软件，但它到底是如何进行文件格式探测呢？

### 首先，每种文件格式或数据包从二进制的角度上理解，都有其独特的数据结构定义

通常很多数据包都是header+payload的结构，header定义了一些元数据（可能是帧率，码率，分辨率等），payload则包含了实际的数据内容

大多数数据包只需要解析header就足以判断它的格式是什么

拿RTP数据包为例，参考RFC 3550，其header结构如下：
```
 0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|X|  CC   |M|     PT      |       sequence number         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                           timestamp                           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           synchronization source (SSRC) identifier            |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |            contributing source (CSRC) identifiers             |
   |                             ....                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

可以看到header中定义了许多元数据，比如版本号、是否有扩展头、负载类型、序列号等。

对每一种文件格式或数据包做数据结构定义的好处，除了存储文件解码所需的数据，更重要的是这种统一的数据结构定义，便于不同的软件（比如ffmpeg）对其进行解析和处理。

所以，FFmpeg就可以根据不同数据包的结构定义，写出对应的probe函数。


## 但是文件格式实在太多了，FFmpeg难免会有一些识别错误的情况

最近有用户反馈一个bug，提供了一个m3u文件:
```
#EXTM3U
./aaaaaaa.00000..aaa
./aaaaaaa.00000..aaa
...
```
是由这行命令生成的
```shell
#EXTM3U
(echo '#EXTM3U'; for i in $(seq 1 102); do echo './aaaaaaa.00000..aaa'; done) >test.m3u
```

这个文件用于mpv的播放列表，但FFmpeg错误的将其识别为AMR文件
```shell
Input #0, amrnb, from 'test.m3u':
  Duration: 00:00:03.31, bitrate: 5 kb/s
  Stream #0:0, 50, 1/8000: Audio: amr_nb (amrnb), 8000 Hz, mono, fltp, 5 kb/s
[AVIOContext @ 0x5563a8503140] Statistics: 2150 bytes read, 0 seeks
```

### 在深入分析probe代码之前，参考RFC 4867, 我们先简单了解一下amr:
1. AMR (Adaptive Multi-Rate)是一种音频编码格式，用于移动网络传输，可以动态调整码率
2. amr有两种存储形式：3GPP（有header）和raw数据（无header），在本例中识别到的是raw 的 amrnb，因此我们只讨论raw格式
3. amrnb是窄带(Narrowband)， amrwb是宽带(Wideband)
4. 在raw amr数据中，每一帧都有一个ToC(Table of Contents)，可以理解为每一帧的header


### 让我们深入分析一下amrnb的probe函数:
```c
static int amrnb_probe(const AVProbeData *p)
{
    int mode, i = 0, valid = 0, invalid = 0;
    const uint8_t *b = p->buf;

    while (i < p->buf_size) {
        mode = b[i] >> 3 & 0x0F;
        if (mode < 9 && (b[i] & 0x4) == 0x4) {
            int last = b[i];
            int size = amrnb_packed_size[mode];
            while (size--) {
                if (b[++i] != last)
                    break;
            }
            if (size > 0) {
                valid++;
                i += size;
            }
        } else {
            valid = 0;
            invalid++;
            i++;
        }
    }
    if (valid > 100 && valid >> 4 > invalid)
        return AVPROBE_SCORE_EXTENSION / 2 + 1;
    return 0;
}
```

由于raw amr数据没有header，只有payload，所以第一个字节就是第一帧的ToC(Table of Contents)，相当于每一帧的header

让我们看一下amrnb ToC的数据结构：
```
  0 1 2 3 4 5 6 7
   +-+-+-+-+-+-+-+-+
   |F|  FT   |Q|P|P|
   +-+-+-+-+-+-+-+-+
```
- `F` 如果为0，表示最后一帧
- `FT` 是 Frame type，表示帧类型（不同类型对应不同码率，对应不同帧大小）
- `Q` 是Frame quality，如果为0，表示帧损坏
- 两个 `P` 是 padding bits，必须为0

### probe函数中核心逻辑如下
```c
    if (mode < 9 && (b[i] & 0x4) == 0x4) {
```
1. ffmpeg检查了ToC中的`FT`和`Q`字段 ，如果符合要求，就认为是一个有效帧，
```c
int size = amrnb_packed_size[mode];
```
2. 然后通过帧类型获取到该帧的大小，跳过该帧，继续检查下一帧。

```c
    if (valid > 100 && valid >> 4 > invalid)
        return AVPROBE_SCORE_EXTENSION / 2 + 1;
```
3. 如果有效帧数量超过100个并且有效帧数量是无效帧数量的16倍以上，就认为是amrnb文件

清楚了probe逻辑后再来看m3u文件
```shell
#EXTM3U
./aaaaaaa.00000..aaa
./aaaaaaa.00000..aaa
...
```

1. 先检查第一个字节 `#`（0010 0011）不是有效帧，跳过第一个字节
2. 检查第二个字节 `E`（0100 0101）， FT解析为8，Q解析为1，符合要求，认为是一个有效帧，根据FT作为index获取该帧大小是6，跳过6个字节
3. 检查第7（1 + 6）个字节 `.` (0010 1110)，FT解析为5，Q解析为1，符合要求，认为是一个有效帧，根据FT获取该帧大小是20，跳过20个字节
4. 检查第27（1 + 6 + 20）个字节 仍然是 `.`于是重复第三步，将所有`./aaaaaaa.00000..aaa`识别为有效帧，最后valid=102, invalid=1，符合要求，认为是amrnb文件

> 根据FT字段，获取到帧大小的列表：
> `amrnb_packed_size[16] = { 13, 14, 16, 18, 20, 21, 27, 32, 6, 1, 1, 1, 1, 1, 1, 1 };`

这个m3u的数据恰到好处，导致其蒙混过关了，想解决这个问题需要仔细看amrnb数据格式的定义，发现FFmpeg目前只分析了`FT`和`Q`字段，而没有分析`F`和`P`字段，其中`F`是0或1均为有效帧，但`P`作为padding bits，必须为0，而上面误判的两个字节`E`(0100 0101)和`.`(0010 1110)的`P`字段不符合全0的定义，所以我们只需要增加对`P`字段的检查即可

修复patch已合并ffmpeg master

https://code.ffmpeg.org/FFmpeg/FFmpeg/commit/ec0173ab59e9927a27a959c8c4706cd5316d0560
