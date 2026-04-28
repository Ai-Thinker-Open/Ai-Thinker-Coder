# Opus 音频编解码器

## 概述

Opus 是一个由 IETF（互联网工程任务组）标准化的交互式语音和音频编解码器。它融合了 Skype 的 SILK 编解码器（针对语音优化）和 Xiph.Org 的 CELT 编解码器（针对音乐优化）的技术，能够在 6 kbps 到 510 kbps 的广泛比特率范围内工作。

Opus 的主要特性包括：

- **采样率支持**：8 kHz 到 48 kHz
- **比特率范围**：6 kbps ~ 510 kbps
- **编码模式**：支持 CBR（恒定比特率）和 VBR（可变比特率）
- **音频带宽**：从窄带（4 kHz）到全频段（20 kHz）
- **声道支持**：单声道、立体声，最多支持 255 个声道
- **帧长度**：2.5 ms 到 60 ms 可调
- **丢包隐藏**：内置 PLC（Packet Loss Concealment）功能
- **双重实现**：提供浮点和定点两种实现

在 BL618 平台上，Opus 主要应用于 VoIP（语音通话）和音乐流媒体场景，能够在网络带宽波动时保持良好的音频质量。

## 版本信息

```c
const char *opus_get_version_string(void);
```

获取 Opus 库的版本字符串。应用程序可以通过检查版本字符串中是否包含 "-fixed" 子串来判断当前是定点版本还是浮点版本构建。

## 数据类型

Opus 使用以下基本数据类型定义在 `opus_types.h` 中：

| 类型 | 说明 |
|------|------|
| `opus_int8` | 有符号 8 位整数 |
| `opus_uint8` | 无符号 8 位整数 |
| `opus_int16` | 有符号 16 位整数 |
| `opus_uint16` | 无符号 16 位整数 |
| `opus_int32` | 有符号 32 位整数 |
| `opus_uint32` | 无符号 32 位整数 |
| `opus_int64` | 有符号 64 位整数 |

## 不透明句柄

Opus 库使用不透明指针作为状态句柄，应用程序不需要了解其内部结构：

| 句柄类型 | 说明 |
|---------|------|
| `OpusEncoder*` | 单流编码器状态句柄 |
| `OpusDecoder*` | 单流解码器状态句柄 |
| `OpusMSEncoder*` | 多流编码器状态句柄 |
| `OpusMSDecoder*` | 多流解码器状态句柄 |
| `OpusRepacketizer*` | 数据包重打包器句柄 |

## 错误码

所有 Opus 函数返回的错误码定义如下：

| 错误码 | 值 | 说明 |
|--------|-----|------|
| `OPUS_OK` | 0 | 操作成功 |
| `OPUS_BAD_ARG` | -1 | 参数无效或超出范围 |
| `OPUS_BUFFER_TOO_SMALL` | -2 | 输出缓冲区空间不足 |
| `OPUS_INTERNAL_ERROR` | -3 | 检测到内部错误 |
| `OPUS_INVALID_PACKET` | -4 | 压缩数据损坏或格式不支持 |
| `OPUS_UNIMPLEMENTED` | -5 | 不支持请求的操作 |
| `OPUS_INVALID_STATE` | -6 | 编码器或解码器结构无效或已释放 |
| `OPUS_ALLOC_FAIL` | -7 | 内存分配失败 |

```c
const char *opus_strerror(int error);
```

将错误码转换为人类可读的错误字符串。

## 编码模式

创建编码器时需要指定应用模式：

```c
#define OPUS_APPLICATION_VOIP                2048
#define OPUS_APPLICATION_AUDIO                2049
#define OPUS_APPLICATION_RESTRICTED_LOWDELAY  2051
```

| 模式 | 说明 |
|------|------|
| `OPUS_APPLICATION_VOIP` | 适用于 VoIP/视频会议，对语音信号进行增强处理，提高可懂度 |
| `OPUS_APPLICATION_AUDIO` | 适用于音乐和广播场景，追求解码音频对原始输入的高保真度 |
| `OPUS_APPLICATION_RESTRICTED_LOWDELAY` | 最低延迟模式，禁用语音优化模式，适用于对延迟要求极高的场景 |

## 编码器 API

### 创建与销毁

```c
OpusEncoder *opus_encoder_create(
    opus_int32 Fs,      // 采样率：8000/12000/16000/24000/48000 Hz
    int channels,       // 声道数：1（单声道）或 2（立体声）
    int application,    // 应用模式
    int *error          // 输出错误码
);

void opus_encoder_destroy(OpusEncoder *st);
```

`opus_encoder_create()` 分配并初始化一个编码器状态。如果创建失败，`error` 将包含错误码，返回值为 `NULL`。

### 编码操作

```c
opus_int32 opus_encode(
    OpusEncoder *st,
    const opus_int16 *pcm,      // 输入 PCM 数据（交叉存储如果为立体声）
    int frame_size,              // 每声道采样数
    unsigned char *data,         // 输出压缩数据缓冲区
    opus_int32 max_data_bytes    // 输出缓冲区最大字节数
);

opus_int32 opus_encode_float(
    OpusEncoder *st,
    const float *pcm,            // 输入浮点 PCM 数据，范围 +/-1.0
    int frame_size,
    unsigned char *data,
    opus_int32 max_data_bytes
);
```

**frame_size 参数说明**（以 48 kHz 为例）：

| 帧时长 | frame_size（采样数） |
|--------|---------------------|
| 2.5 ms | 120 |
| 5 ms | 240 |
| 10 ms | 480 |
| 20 ms | 960 |
| 40 ms | 1920 |
| 60 ms | 2880 |

返回值为压缩数据包的实际字节数，返回值 ≤ 2 表示可以启用 DTX（不传输）。返回负值表示编码错误。

### 编码器控制

```c
int opus_encoder_ctl(OpusEncoder *st, int request, ...);
```

通过 CTL 接口可以动态调整编码器参数。常用控制宏：

```c
// 设置比特率（bps）
opus_encoder_ctl(enc, OPUS_SET_BITRATE(bitrate));
opus_encoder_ctl(enc, OPUS_GET_BITRATE(&bitrate));

// 设置应用模式
opus_encoder_ctl(enc, OPUS_SET_APPLICATION(OPUS_APPLICATION_VOIP));
opus_encoder_ctl(enc, OPUS_GET_APPLICATION(&app));

// 设置采样率
opus_encoder_ctl(enc, OPUS_SET_SAMPLE_RATE(sample_rate));

// 设置复杂度（0-10，默认为 10）
opus_encoder_ctl(enc, OPUS_SET_COMPLEXITY(10));

// 启用/禁用 VBR
opus_encoder_ctl(enc, OPUS_SET_VBR(1));  // 1=启用 VBR，0=禁用

// 设置信号类型提示
opus_encoder_ctl(enc, OPUS_SET_SIGNAL(OPUS_SIGNAL_VOICE));
// OPUS_SIGNAL_MUSIC - 偏向音乐
// OPUS_SIGNAL_VOICE - 偏向语音
// OPUS_AUTO - 自动选择（默认）
```

## 解码器 API

### 创建与销毁

```c
OpusDecoder *opus_decoder_create(
    opus_int32 Fs,      // 采样率：8000/12000/16000/24000/48000 Hz
    int channels,       // 声道数
    int *error          // 输出错误码
);

void opus_decoder_destroy(OpusDecoder *st);
```

### 解码操作

```c
int opus_decode(
    OpusDecoder *st,
    const unsigned char *data,   // 输入压缩数据包
    opus_int32 len,              // 数据包字节数，0 表示丢包
    opus_int16 *pcm,             // 输出 PCM 缓冲区
    int frame_size,              // 每声道输出采样数
    int decode_fec               // 是否解码前向纠错数据（0/1）
);

int opus_decode_float(
    OpusDecoder *st,
    const unsigned char *data,
    opus_int32 len,
    float *pcm,
    int frame_size,
    int decode_fec
);
```

**丢包处理**：当 `data` 为 `NULL` 且 `len` 为 0 时，执行 PLC 丢包隐藏，生成与丢失帧相同长度的音频。

** FEC 处理**：当 `decode_fec` 为 1 时，尝试解码包中嵌入的前向纠错数据。

### 解码器控制

```c
int opus_decoder_ctl(OpusDecoder *st, int request, ...);
```

常用解码器 CTL：

```c
// 获取采样率
opus_int32 sample_rate;
opus_decoder_ctl(dec, OPUS_GET_SAMPLE_RATE(&sample_rate));

// 获取前一包的持续时间
opus_decoder_ctl(dec, OPUS_GET_LAST_PACKET_DURATION(&duration));
```

## 数据包分析 API

```c
int opus_packet_get_bandwidth(const unsigned char *data);
```

获取 Opus 数据包的音频带宽：

| 返回值 | 带宽 |
|--------|------|
| `OPUS_BANDWIDTH_NARROWBAND` | 4 kHz（窄带） |
| `OPUS_BANDWIDTH_MEDIUMBAND` | 6 kHz（中等带宽） |
| `OPUS_BANDWIDTH_WIDEBAND` | 8 kHz（宽带） |
| `OPUS_BANDWIDTH_SUPERWIDEBAND` | 12 kHz（超宽带） |
| `OPUS_BANDWIDTH_FULLBAND` | 20 kHz（全频段） |

```c
int opus_packet_get_nb_channels(const unsigned char *data);
```

获取 Opus 数据包中的声道数。

## 多流编码器/解码器

对于需要处理超过 2 个声道的应用，可以使用多流 API：

```c
OpusMSEncoder *opus_multistream_encoder_create(
    opus_int32 Fs,
    int channels,           // 总声道数
    int streams,           // 编码流数量
    int coupled_streams,   // 立体声流数量
    const unsigned char *mapping,
    int application,
    int *error
);

OpusMSDecoder *opus_multistream_decoder_create(
    opus_int32 Fs,
    int channels,
    int streams,
    int coupled_streams,
    const unsigned char *mapping,
    int *error
);
```

## 代码示例

以下示例展示在 BL618 上使用 Opus 进行 PCM 数据编码和解码的基本流程：

### 初始化

```c
#include "opus.h"
#include "opus_defines.h"
#include <stdio.h>
#include <stdlib.h>

// 错误检查辅助宏
#define CHECK_ERROR(ret, ctx) do { \
    if ((ret) != OPUS_OK) { \
        fprintf(stderr, "Opus error: %s\n", opus_strerror(ret)); \
        goto ctx; \
    } \
} while(0)

// 采样率
#define SAMPLE_RATE     48000
// 声道数
#define CHANNELS        1
// 帧时长（20 ms）
#define FRAME_SIZE      (SAMPLE_RATE * 20 / 1000)  // 960 samples
// 最大数据包大小
#define MAX_PACKET_SIZE 4000
```

### 编码器示例

```c
int opus_encode_example(const opus_int16 *pcm_data, int frame_size,
                        unsigned char *opus_packet, opus_int32 *packet_size)
{
    OpusEncoder *encoder = NULL;
    int error;
    opus_int32 len;

    // 创建编码器（VoIP 模式）
    encoder = opus_encoder_create(SAMPLE_RATE, CHANNELS,
                                   OPUS_APPLICATION_VOIP, &error);
    CHECK_ERROR(error, cleanup);

    // 可选：设置编码参数
    opus_encoder_ctl(encoder, OPUS_SET_BITRATE(64000));  // 64 kbps
    opus_encoder_ctl(encoder, OPUS_SET_COMPLEXITY(10));  // 最高复杂度
    opus_encoder_ctl(encoder, OPUS_SET_VBR(1));           // 启用 VBR

    // 执行编码
    len = opus_encode(encoder, pcm_data, frame_size,
                      opus_packet, MAX_PACKET_SIZE);
    if (len < 0) {
        fprintf(stderr, "Encode failed: %s\n", opus_strerror(len));
        error = len;
        goto cleanup;
    }

    *packet_size = len;
    error = OPUS_OK;

cleanup:
    if (encoder) {
        opus_encoder_destroy(encoder);
    }
    return error;
}
```

### 解码器示例

```c
int opus_decode_example(const unsigned char *opus_packet, opus_int32 packet_size,
                        opus_int16 *pcm_data, int frame_size)
{
    OpusDecoder *decoder = NULL;
    int error;
    int samples;

    // 创建解码器
    decoder = opus_decoder_create(SAMPLE_RATE, CHANNELS, &error);
    CHECK_ERROR(error, cleanup);

    // 执行解码
    // decode_fec = 0 表示不解码前向纠错数据
    samples = opus_decode(decoder, opus_packet, packet_size,
                           pcm_data, frame_size, 0);
    if (samples < 0) {
        fprintf(stderr, "Decode failed: %s\n", opus_strerror(samples));
        error = samples;
        goto cleanup;
    }

    error = OPUS_OK;

cleanup:
    if (decoder) {
        opus_decoder_destroy(decoder);
    }
    return error;
}
```

### 丢包隐藏示例

```c
int opus_plc_example(OpusDecoder *decoder, opus_int16 *pcm_data, int frame_size)
{
    int samples;

    // 传入 NULL 和 0 表示丢包，触发 PLC
    samples = opus_decode(decoder, NULL, 0, pcm_data, frame_size, 0);
    if (samples < 0) {
        fprintf(stderr, "PLC failed: %s\n", opus_strerror(samples));
        return samples;
    }

    return samples;
}
```

### 完整编码解码流程

```c
int opus_full_example(const opus_int16 *input_pcm, int frame_size,
                      opus_int16 *output_pcm, int *output_size)
{
    OpusEncoder *encoder = NULL;
    OpusDecoder *decoder = NULL;
    unsigned char packet[MAX_PACKET_SIZE];
    opus_int32 packet_size;
    int error = OPUS_OK;
    int decoded_samples;

    // 创建编码器
    encoder = opus_encoder_create(SAMPLE_RATE, CHANNELS,
                                  OPUS_APPLICATION_VOIP, &error);
    if (error != OPUS_OK) goto cleanup;

    // 创建解码器
    decoder = opus_decoder_create(SAMPLE_RATE, CHANNELS, &error);
    if (error != OPUS_OK) goto cleanup;

    // 编码：PCM -> Opus 数据包
    packet_size = opus_encode(encoder, input_pcm, frame_size,
                               packet, MAX_PACKET_SIZE);
    if (packet_size < 0) {
        error = packet_size;
        goto cleanup;
    }

    // 解码：Opus 数据包 -> PCM
    decoded_samples = opus_decode(decoder, packet, packet_size,
                                   output_pcm, frame_size, 0);
    if (decoded_samples < 0) {
        error = decoded_samples;
        goto cleanup;
    }

    *output_size = decoded_samples;

cleanup:
    if (encoder) opus_encoder_destroy(encoder);
    if (decoder) opus_decoder_destroy(decoder);
    return error;
}
```

## 性能注意事项

1. **帧大小选择**：较长的帧（如 20 ms、40 ms、60 ms）能提供更好的压缩效率，但会增加延迟。对于 VoIP 应用，推荐使用 20 ms 帧。

2. **复杂度设置**：`OPUS_SET_COMPLEXITY` 的值越高，编码质量越好，但 CPU 消耗也越高。在 BL618 等嵌入式平台上，可根据性能需求选择合适的复杂度（建议 1-8）。

3. **VBR vs CBR**：VBR（可变比特率）能在保持质量的同时节省平均带宽，但输出数据大小不固定。CBR 适合对带宽有严格要求的场景。

4. **丢包保护**：启用带内 FEC（`OPUS_SET_INBAND_FEC(1)`）会增加少量额外带宽，但能提高丢包率较高网络下的音质。

5. **内存占用**：编码器和解码器状态需要持续内存，销毁后立即释放。

## 参考

- [RFC 6716](https://tools.ietf.org/html/rfc6716) - Opus Audio Codec
- [RFC 7845](https://tools.ietf.org/html/rfc7845) - Ogg Encapsulation for Opus Audio
- Xiph.Org Foundation: <https://opus-codec.org/>
- BL618 Bouffalo SDK Opus 组件源码：`components/multimedia/opus/include/opus/`
