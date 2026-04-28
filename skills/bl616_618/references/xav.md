# XAV 多媒体框架

## 概述

XAV 是 BL618 芯片的多媒体处理框架，提供完整的音视频播放、编解码、封装/解封装、音频滤波解决方案。该框架采用模块化设计，遵循 `player -> avformat(demux) -> avcodec(dec) -> avfilter -> ao` 的数据流 pipeline架构，能够处理本地文件和网络流等多种媒体源。

XAV 框架支持 AAC、MP3、OPUS、AMR 等主流音频编解码格式，以及 MP4、MKV、TS、FLV、OGG 等封装容器。开发者可通过统一的 API 接口快速实现多媒体播放功能，适用于智能音箱、物联网设备、音频播放器等多种应用场景。

## 框架结构

XAV 框架各组件协同工作，构成完整的多媒体处理 pipeline：

```
┌─────────┐    ┌───────────┐    ┌──────────┐    ┌───────────┐    ┌─────┐
│ player  │───▶│  avformat │───▶│ avcodec  │───▶│ avfilter  │───▶│ ao  │
└─────────┘    │  (demux)  │    │  (dec)   │    │ (EQ/Vol)  │    └─────┘
               └───────────┘    └──────────┘    └───────────┘
                     │
               ┌─────┴─────┐
               │  stream   │
               │(file/socket/mem)
               └───────────┘
```

- **player**: 高级播放器封装，提供播放控制接口
- **stream**: 数据流抽象层，支持文件、网络、内存等数据源
- **avformat**: 封装格式处理，负责解封装(demux)和封装(mux)
- **avcodec**: 音视频编解码抽象层
- **avfilter**: 音频滤波器，处理 EQ、音量、变速等效果
- **ao**: 音频输出设备抽象

## 组件详解

### 1. xplayer - 多媒体播放器

xplayer 是 XAV 框架的高级播放器接口，提供完整的播放控制功能。支持本地文件播放（如 `file:///fatfs0/Music/1.mp3`）和网络流播放（如 `http://ip:port/1.mp3`）。

#### 播放器生命周期管理

| 函数 | 说明 |
|------|------|
| `xplayer_module_init()` | 初始化播放器模块（全局初始化，仅调用一次） |
| `xplayer_module_config_init()` | 初始化播放器模块配置参数 |
| `xplayer_new()` | 创建播放器实例，name 为 NULL 时创建默认播放器 |
| `xplayer_free()` | 销毁播放器实例 |

#### 播放控制接口

| 函数 | 说明 |
|------|------|
| `xplayer_set_url()` | 设置媒体源 URL |
| `xplayer_play()` | 开始播放 |
| `xplayer_pause()` | 暂停播放 |
| `xplayer_resume()` | 恢复播放 |
| `xplayer_stop()` | 停止播放 |
| `xplayer_seek()` | 跳转到指定时间位置（timestamp 单位为毫秒） |
| `xplayer_set_start_time()` | 设置播放起始时间 |

#### 状态与信息获取

| 函数 | 说明 |
|------|------|
| `xplayer_get_time()` | 获取当前播放时间和总时长 |
| `xplayer_get_media_info()` | 获取媒体信息（需调用 `media_info_uninit` 释放） |
| `xplayer_get_vol()` | 获取音量值（0~255） |
| `xplayer_set_vol()` | 设置音量值（0~255） |
| `xplayer_get_speed()` | 获取播放速度 |
| `xplayer_set_speed()` | 设置播放速度（建议范围 0.5 ~ 2.0） |
| `xplayer_get_url()` | 获取当前媒体 URL |
| `xplayer_get_state()` | 获取播放器状态 |

#### 事件回调

```c
int xplayer_set_callback(xplayer_t *xplayer, xplayer_eventcb_t event_cb, const void *user_data);
```

设置播放器事件回调，在播放开始、暂停、停止、播放完成等事件时通知应用层。

#### 配置管理

```c
int xplayer_set_config(xplayer_t *xplayer, const xplayer_cnf_t *conf);
int xplayer_get_config(xplayer_t *xplayer, xplayer_cnf_t *conf);
```

在播放前设置播放器的配置参数，如缓存大小、播放模式等。

### 2. avcodec - 音视频编解码抽象层

avcodec 模块提供音视频编解码的统一抽象接口，支持多种音频编解码器。

#### 音频解码器管理

| 函数 | 说明 |
|------|------|
| `ad_ops_register()` | 注册解码器操作接口 |
| `ad_conf_init()` | 初始化解码器配置参数 |
| `ad_open()` | 打开/创建音频解码器实例 |
| `ad_close()` | 关闭/销毁解码器实例 |
| `ad_reset()` | 重置解码器状态 |

#### 解码操作

| 函数 | 说明 |
|------|------|
| `ad_decode()` | 解码一帧音频数据 |
| `ad_control()` | 控制解码器（如设置参数、获取状态） |

```c
// 解码接口说明
int ad_decode(ad_cls_t *o, avframe_t *frame, int *got_frame, const avpacket_t *pkt);
```

- `o`: 解码器实例
- `frame`: 输出 PCM 数据帧
- `got_frame`: 输出参数，表示是否成功解码出一帧
- `pkt`: 输入的编码数据包
- 返回值: 成功解码时返回消耗的输入字节数，失败返回 -1

#### 支持的音频编解码格式

- **AAC**: 高级音频编码，支持多种采样率和声道配置
- **MP3**: MPEG-1 Audio Layer III，广泛应用的音频格式
- **OPUS**: 高效音频编解码器，适合网络传输
- **AMR**: 自适应多速率编码，常用于语音场景

#### 解码器配置参数

```c
typedef struct ad_conf {
    sf_t          sf;             // 采样格式
    uint8_t       *extradata;     // 额外数据（如 ADTS 头）
    int32_t       extradata_size;
    uint32_t      block_align;    // 每帧数据大小
    uint32_t      bps;            // 比特率
} ad_conf_t;
```

### 3. avformat - 封装格式处理

avformat 模块负责处理各种媒体封装格式，实现解封装（demux）和封装（mux）功能。

#### 核心接口

| 函数 | 说明 |
|------|------|
| `demux_ops_register()` | 注册解封装器操作接口 |
| `demux_open()` | 打开解封装器，参数为 stream 句柄 |
| `demux_read_packet()` | 读取一个媒体数据包（音视频帧） |
| `demux_seek()` | 跳转到指定时间位置（毫秒） |
| `demux_close()` | 关闭解封装器 |
| `demux_control()` | 控制解封装器 |

#### 数据包读取

```c
int demux_read_packet(demux_cls_t *o, avpacket_t *pkt);
```

从媒体文件中读取一个压缩的音视频数据包，返回后需送入对应的解码器进行解码。

#### 支持的封装格式

| 容器格式 | 说明 |
|----------|------|
| MP4 | 基于 ISO Base Media File Format，常用于 H.264/AAC 编码 |
| MKV | Matroska 多媒体容器，支持多音轨、多字幕 |
| TS | MPEG-2 Transport Stream，用于直播流 |
| FLV | Flash Video，用于 RTMP 流媒体 |
| OGG | 开源多媒体容器，常用于 Vorbis/Opus 音频 |

### 4. avfilter - 音频滤波器

avfilter 模块提供音频信号处理能力，包括均衡器（EQ）、音量控制、播放速度调整等。

#### 滤波器接口

| 函数 | 说明 |
|------|------|
| `avf_init()` | 初始化滤波器实例 |
| `avf_control()` | 控制滤波器（设置参数） |
| `avf_set_bypass()` | 设置是否旁路滤波器 |
| `avf_link()` | 链接两个滤波器构成滤波器链 |
| `avf_link_tail()` | 将尾滤波器链接到链首 |
| `avf_filter_frame()` | 对音频帧进行滤波处理 |
| `avf_uninit()` | 释放滤波器内部资源 |
| `avf_close()` | 关闭并释放滤波器 |
| `avf_chain_close()` | 关闭整个滤波器链 |

#### 音频帧处理

```c
int avf_filter_frame(avfilter_t *avf, const avframe_t *in, avframe_t *out);
```

对输入音频帧 `in` 进行滤波处理后输出到 `out`。返回值为每通道输出的采样数，出错返回 -1。

#### 滤波器链示例

```
输入帧 → [EQ Filter] → [Volume Filter] → [atempo Filter] → 输出帧
```

通过 `avf_link()` 可以将多个滤波器串联成链，实现复杂的音频处理流程。

#### 支持的滤波器类型

- **EQ (均衡器)**: 调整不同频段的增益
- **Volume (音量)**: 音量增益/衰减控制
- **atempo (变速)**: 调整播放速度而不改变音高
- **swr (重采样)**: 音频格式转换

### 5. stream - 数据流抽象

stream 模块提供统一的数据流抽象接口，支持文件、网络、内存等多种数据源。

#### 流操作接口

| 函数 | 说明 |
|------|------|
| `stream_ops_register()` | 注册流操作接口 |
| `stream_conf_init()` | 初始化流配置参数 |
| `stream_open()` | 打开数据流 |
| `stream_read()` | 从流读取数据 |
| `stream_write()` | 向流写入数据 |
| `stream_seek()` | 流定位（SEEK_SET/SEEK_CUR/SEEK_END） |
| `stream_skip()` | 跳过指定字节数 |
| `stream_tell()` | 获取当前流位置 |
| `stream_get_size()` | 获取流大小 |
| `stream_get_url()` | 获取流 URL |
| `stream_close()` | 关闭流 |
| `stream_control()` | 控制流 |
| `stream_is_seekable()` | 判断流是否支持定位 |
| `stream_is_eof()` | 判断流是否结束 |
| `stream_is_live()` | 判断是否为直播流 |
| `stream_is_interrupt()` | 检查流是否被中断 |

#### 流配置参数

```c
typedef struct stream_conf {
    enum stream_mode    mode;               // 流模式
    irq_av_t            irq;                // 中断处理
    uint8_t             need_parse;         // 是否需要解析 URL
    uint32_t            rcv_timeout;         // 接收超时（毫秒）
    uint32_t            cache_size;          // 缓存大小
    uint32_t            cache_start_threshold; // 缓存启动阈值 (0~100)
    get_decrypt_cb_t    get_dec_cb;         // 解密回调
    void                *opaque;            // 用户数据
    stream_event_t      stream_event_cb;     // 流事件回调
} stm_conf_t;
```

#### 支持的 URL 前缀

| 前缀 | 说明 |
|------|------|
| `file://` | 本地文件，如 `file:///fatfs0/Music/1.mp3` |
| `http://` | HTTP 网络流 |
| `https://` | HTTPS 加密网络流 |
| `mem://` | 内存数据源，如 `mem://addr=%u&size=%u` |
| `fifo://` | FIFO 管道，如 `fifo://tts/1` |

#### 二进制读取接口

stream 模块提供便捷的二进制数据读取接口：

```c
int stream_r8(stream_cls_t *o);
uint16_t stream_r16be(stream_cls_t *o);
uint16_t stream_r16le(stream_cls_t *o);
uint32_t stream_r24be(stream_cls_t *o);
uint32_t stream_r24le(stream_cls_t *o);
uint32_t stream_r32be(stream_cls_t *o);
uint32_t stream_r32le(stream_cls_t *o);
uint64_t stream_r64be(stream_cls_t *o);
uint64_t stream_r64le(stream_cls_t *o);
```

支持大端(be)和小端(le)格式的整数读取。

### 6. ao - 音频输出设备抽象

ao 模块提供音频输出设备的统一抽象，支持配置音频参数、缓存管理和音量控制。

#### 音频输出接口

| 函数 | 说明 |
|------|------|
| `ao_ops_register()` | 注册音频输出操作接口 |
| `ao_conf_init()` | 初始化音频输出配置参数 |
| `ao_open()` | 打开音频输出设备 |
| `ao_start()` | 启动音频输出 |
| `ao_stop()` | 停止音频输出 |
| `ao_write()` | 写入音频数据 |
| `ao_drain()` | 排空尾部 PCM 数据 |
| `ao_close()` | 关闭音频输出设备 |
| `ao_control()` | 控制音频输出 |

#### 音频输出配置

```c
typedef struct ao_conf {
    char        *name;           // 音频输出名称
    uint32_t    period_ms;       // 周期缓存大小（毫秒）
    uint32_t    period_num;      // 周期数量
    uint8_t     eq_segments;     // 均衡器段数
    uint8_t     *aef_conf;       // AEF 配置数据
    size_t      aef_conf_size;   // AEF 配置数据大小
    uint32_t    resample_rate;   // 重采样目标采样率（非零表示需要重采样）
    uint8_t     vol_en;          // 软件音量使能
    uint8_t     vol_index;       // 软件音量值（0~255）
    uint8_t     atempo_play_en;  // 变速播放使能
    float       speed;           // 播放速度（建议 0.5 ~ 2.0）
    int32_t     db_min;          // 数字音量曲线最小值
    int32_t     db_max;          // 数字音量曲线最大值
} ao_conf_t;
```

### 7. swresample - 重采样与格式转换

swresample 模块提供音频重采样和格式转换功能。

#### 采样率转换

| 函数 | 说明 |
|------|------|
| `resx_ops_register()` | 注册重采样操作接口 |
| `resx_new()` | 创建重采样器实例 |
| `resx_get_osamples_max()` | 获取最大输出采样数 |
| `resx_convert()` | 执行重采样转换 |
| `resx_free()` | 释放重采样器 |

```c
resx_t *resx_new(uint32_t irate, uint32_t orate, uint8_t channels, uint8_t bits);
```

创建重采样器：输入采样率 `irate`、输出采样率 `orate`、声道数 `channels`、位深 `bits`（仅支持 16 位）。

#### 采样格式转换

swresample 提供 `swr_convert` 和 `swr_convert_frame` 接口进行采样格式转换：

```c
swr_t *swr_new(sf_t isf, sf_t osf);
int swr_convert(swr_t *s, void **out, size_t nb_osamples, const void **in, size_t nb_isamples);
int swr_convert_frame(swr_t *s, const avframe_t *iframe, avframe_t *oframe);
void swr_free(swr_t *s);
```

#### PCM 格式转换宏

pcm_convert 模块提供各种 PCM 格式之间的转换函数：

| 函数 | 说明 |
|------|------|
| `s8_ch1_to_s16_ch2()` | 单声道 S8 转立体声 S16 |
| `s8_ch1_to_s16_ch1()` | 单声道 S8 转单声道 S16 |
| `u8_ch1_to_s16_ch2()` | 单声道 U8 转立体声 S16 |
| `s8_ch2_to_s16_ch2()` | 立体声 S8 转立体声 S16 |
| `s8_ch2_to_s16_ch1()` | 立体声 S8 转单声道 S16 |
| `s16_ch1_to_s16_ch2()` | 单声道 S16 转立体声 S16 |
| `s16_ch2_to_s16_ch1()` | 立体声 S16 转单声道 S16 |
| `u16_le_to_s16_le()` | U16 小端转 S16 小端 |
| `u16_be_to_s16_le()` | U16 大端转 S16 小端 |

## 与 Camera/Display 协同

XAV 框架支持与摄像头（camera）和显示（display）模块协同工作，实现音视频同步播放。

### 视频显示控制

播放器提供以下视频显示控制接口：

| 函数 | 说明 |
|------|------|
| `xplayer_set_video_visible()` | 显示/隐藏视频 |
| `xplayer_set_display_window()` | 设置显示窗口 |
| `xplayer_set_fullscreen()` | 设置全屏/窗口模式 |
| `xplayer_set_display_format()` | 设置显示格式 |
| `xplayer_set_video_rotate()` | 旋转视频 |
| `xplayer_set_video_crop()` | 裁剪视频 |

### 音视频同步

在播放视频时，框架内部会自动处理音视频同步。音频流通过 ao 输出，视频帧通过 display 模块渲染，两者由播放器统一调度保持同步。

### 多媒体轨道切换

| 函数 | 说明 |
|------|------|
| `xplayer_switch_audio_track()` | 切换音轨 |
| `xplayer_switch_subtitle_track()` | 切换字幕轨道 |
| `xplayer_set_subtitle_url()` | 设置外部字幕 URL |
| `xplayer_set_subtitle_visible()` | 显示/隐藏字幕 |

## 代码示例

### 播放本地 MP3 文件

以下示例展示如何使用 xplayer 播放本地 MP3 文件：

```c
#include "xplayer/xplayer.h"
#include "avutil/av_config.h"

// 播放器事件回调
static void player_event_cb(xplayer_event_t event, void *user_data)
{
    switch (event) {
        case XPLAYER_EVENT_START:
            printf("Playback started\n");
            break;
        case XPLAYER_EVENT_PAUSE:
            printf("Playback paused\n");
            break;
        case XPLAYER_EVENT_STOP:
            printf("Playback stopped\n");
            break;
        case XPLAYER_EVENT-finISH:
            printf("Playback finished\n");
            break;
        case XPLAYER_EVENT_ERROR:
            printf("Playback error\n");
            break;
        default:
            break;
    }
}

int main(void)
{
    xplayer_t *player;
    xplayer_mdl_cnf_t mdl_cnf;
    
    // 1. 初始化播放器模块
    xplayer_module_config_init(&mdl_cnf);
    if (xplayer_module_init(&mdl_cnf) < 0) {
        printf("Failed to init player module\n");
        return -1;
    }
    
    // 2. 创建播放器实例
    player = xplayer_new(NULL);
    if (player == NULL) {
        printf("Failed to create player\n");
        return -1;
    }
    
    // 3. 设置事件回调
    xplayer_set_callback(player, player_event_cb, NULL);
    
    // 4. 设置媒体源
    if (xplayer_set_url(player, "file:///fatfs0/Music/test.mp3") < 0) {
        printf("Failed to set URL\n");
        xplayer_free(player);
        return -1;
    }
    
    // 5. 设置音量 (0~255)
    xplayer_set_vol(player, 200);
    
    // 6. 开始播放
    if (xplayer_play(player) < 0) {
        printf("Failed to start playback\n");
        xplayer_free(player);
        return -1;
    }
    
    // 7. 获取播放信息
    xplay_time_t time;
    xplayer_get_time(player, &time);
    printf("Duration: %llu ms, Current: %llu ms\n", time.duration, time.current_time);
    
    // 8. 播放控制示例
    sleep(5);  // 播放 5 秒
    
    // 暂停
    xplayer_pause(player);
    sleep(2);  // 暂停 2 秒
    
    // 恢复
    xplayer_resume(player);
    
    // 跳转至 30 秒位置
    xplayer_seek(player, 30000);
    
    // 9. 停止播放并释放资源
    xplayer_stop(player);
    xplayer_free(player);
    
    return 0;
}
```

### 使用 avcodec 解码音频

以下示例展示如何手动解码音频数据：

```c
#include "avcodec/ad.h"
#include "avformat/demux.h"
#include "stream/stream.h"
#include "output/ao.h"

// 解码播放流程示例
int decode_and_play(const char *url)
{
    stream_cls_t *stream;
    demux_cls_t *demux;
    ad_cls_t *decoder;
    ao_cls_t *ao;
    ad_conf_t ad_cnf = {0};
    ao_conf_t ao_cnf = {0};
    avpacket_t pkt;
    avframe_t frame;
    int got_frame;
    
    // 1. 打开流
    stream = stream_open(url, NULL);
    if (stream == NULL) {
        return -1;
    }
    
    // 2. 打开解封装器
    demux = demux_open(stream);
    if (demux == NULL) {
        stream_close(stream);
        return -1;
    }
    
    // 3. 打开音频解码器 (假设为 AAC)
    ad_cnf.sf = SF_S16_STEREO;
    decoder = ad_open(AVCODEC_ID_AAC, &ad_cnf);
    if (decoder == NULL) {
        demux_close(demux);
        stream_close(stream);
        return -1;
    }
    
    // 4. 打开音频输出
    ao_cnf.period_ms = 50;
    ao_cnf.vol_en = 1;
    ao_cnf.vol_index = 200;
    ao = ao_open(SF_S16_STEREO, &ao_cnf);
    if (ao == NULL) {
        ad_close(decoder);
        demux_close(demux);
        stream_close(stream);
        return -1;
    }
    
    ao_start(ao);
    
    // 5. 解码循环
    while (1) {
        // 读取压缩数据包
        if (demux_read_packet(demux, &pkt) < 0) {
            break;
        }
        
        // 解码
        int ret = ad_decode(decoder, &frame, &got_frame, &pkt);
        if (ret < 0 || !got_frame) {
            continue;
        }
        
        // 输出音频
        ao_write(ao, frame.data, frame.size);
    }
    
    // 6. 清理资源
    ao_drain(ao);
    ao_close(ao);
    ad_close(decoder);
    demux_close(demux);
    stream_close(stream);
    
    return 0;
}
```

## 数据结构

### 关键数据类型

| 类型 | 说明 |
|------|------|
| `xplayer_t` | 播放器句柄 |
| `stream_cls_t` | 流句柄 |
| `demux_cls_t` | 解封装器句柄 |
| `ad_cls_t` | 音频解码器句柄 |
| `avfilter_t` | 音频滤波器句柄 |
| `ao_cls_t` | 音频输出句柄 |
| `swr_t` | 软件重采样器句柄 |
| `resx_t` | 重采样器句柄 |
| `avframe_t` | 音频帧数据结构 |
| `avpacket_t` | 音视频数据包结构 |

### 播放器状态

```
xplayer 状态枚举:
- XPLAYER_STATE_IDLE      // 空闲状态
- XPLAYER_STATE_PREPARE   // 准备状态
- XPLAYER_STATE_PLAYING   // 播放中
- XPLAYER_STATE_PAUSED    // 已暂停
- XPLAYER_STATE_STOPPED   // 已停止
```

## 错误处理

XAV 框架所有接口遵循统一的错误处理规范：

- 返回值 `0` 表示成功
- 返回值 `-1` 或负值表示失败
- 使用具体错误码时，正值表示成功相关含义（如 `ad_decode` 返回消耗的字节数）

建议错误处理模式：

```c
int ret = xplayer_play(player);
if (ret < 0) {
    printf("Player start failed: %d\n", ret);
    // 错误处理
    return ret;
}
```

## 参考

- [Bouffalo SDK 官方文档](../CLAUDE.md)
- 多媒体组件源码: `components/multimedia/xav/`
- xplayer 接口定义: `include/xplayer/xplayer.h`
- avcodec 接口定义: `include/avcodec/avcodec.h`
- avformat 接口定义: `include/avformat/avformat.h`
- stream 接口定义: `include/stream/stream.h`
- avfilter 接口定义: `include/avfilter/avfilter.h`
- ao 接口定义: `include/output/ao.h`
- swresample 接口定义: `include/swresample/swresample.h`
