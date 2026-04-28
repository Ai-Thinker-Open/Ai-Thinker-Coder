# Vorbis 音频编解码器 API 参考

## 概述

Vorbis 是 Xiph.Org 开发的开源有损音频编解码器，类似于 MP3，但采用更先进的压缩算法。Vorbis 音频通常封装在 OGG 容器中（又称 OggVorbis），广泛应用于游戏、音乐流媒体和网络音频传输领域。在 BL618 芯片平台上，Vorbis 解码库可用于实现音乐播放功能，支持从文件系统或流媒体源解码播放 OGG/Vorbis 音频。

## 关键数据类型

### OggVorbis_File

OggVorbis_File 是最重要的文件句柄结构体，用于管理一个 OGG/Vorbis 文件的完整状态：

```c
typedef struct OggVorbis_File {
  void            *datasource;   // 数据源指针（FILE* 或自定义）
  int              seekable;      // 是否支持 seek
  ogg_int64_t      offset;        // 当前文件偏移
  ogg_int64_t      end;           // 文件结束位置
  ogg_sync_state   oy;            // OGG 同步状态
  int              links;         // 逻辑流数量
  ogg_int64_t     *offsets;       // 各流偏移表
  ogg_int64_t     *dataoffsets;   // 各流数据偏移
  long            *serialnos;     // 串行号列表
  ogg_int64_t     *pcmlengths;    // PCM 采样总数
  vorbis_info     *vi;            // 音频流信息
  vorbis_comment  *vc;            // 用户注释信息
  ogg_int64_t      pcm_offset;    // 当前 PCM 位置
  int              ready_state;   // 就绪状态
  long             current_serialno;
  int              current_link;
  double           bittrack;
  double           samptrack;
  ogg_stream_state os;            // OGG 流状态
  vorbis_dsp_state vd;            // DSP 解码状态
  vorbis_block     vb;            // 音频块工作区
  ov_callbacks     callbacks;      // 回调函数集
} OggVorbis_File;
```

### vorbis_info

vorbis_info 存储音频流的基本配置信息：

```c
typedef struct vorbis_info{
  int version;           // Vorbis 版本
  int channels;          // 声道数（1=单声道，2=立体声）
  long rate;             // 采样率（Hz）
  long bitrate_upper;    // 最高码率
  long bitrate_nominal;  // 标称码率
  long bitrate_lower;    // 最低码率
  long bitrate_window;   // 码率窗口
  void *codec_setup;     // 编解码器内部配置
} vorbis_info;
```

### vorbis_comment

vorbis_comment 存储音频文件的元数据注释（类似 ID3 标签）：

```c
typedef struct vorbis_comment{
  char **user_comments;    // 注释数组
  int   *comment_lengths; // 各注释长度
  int    comments;        // 注释数量
  char  *vendor;          // 编码器供应商字符串
} vorbis_comment;
```

### ov_callbacks

ov_callbacks 定义文件操作的回调函数集，兼容 stdio 接口：

```c
typedef struct {
  size_t (*read_func)  (void *ptr, size_t size, size_t nmemb, void *datasource);
  int    (*seek_func)  (void *datasource, ogg_int64_t offset, int whence);
  int    (*close_func) (void *datasource);
  long   (*tell_func)  (void *datasource);
} ov_callbacks;
```

预定义的回调集：
- `OV_CALLBACKS_DEFAULT`：标准文件操作（可 seek/close）
- `OV_CALLBACKS_NOCLOSE`：不关闭数据源
- `OV_CALLBACKS_STREAMONLY`：流模式（不可 seek）
- `OV_CALLBACKS_STREAMONLY_NOCLOSE`：流模式且不关闭

### 不透明结构体

以下结构体为不透明类型，仅通过 API 函数操作：

- **vorbis_dsp_state**：DSP 状态结构，缓冲当前音频分析/合成状态
- **vorbis_block**：音频块结构，单个待处理的音频数据块

## 核心 API

### 文件打开与关闭

#### ov_open_callbacks()

```c
int ov_open_callbacks(void *datasource, OggVorbis_File *vf,
                      const char *initial, long ibytes, ov_callbacks callbacks);
```

通过自定义回调函数打开 OGG/Vorbis 文件。datasource 可以是 FILE* 或其他自定义数据源。initial 和 ibytes 用于跳过文件头部（通常传入 NULL 和 0）。返回 0 表示成功。

#### ov_fopen()

```c
int ov_fopen(const char *path, OggVorbis_File *vf);
```

便捷函数，直接通过文件路径打开文件，内部使用标准文件操作回调。

#### ov_test_callbacks() / ov_test()

```c
int ov_test_callbacks(void *datasource, OggVorbis_File *vf,
                     const char *initial, long ibytes, ov_callbacks callbacks);
int ov_test(FILE *f, OggVorbis_File *vf, const char *initial, long ibytes);
```

测试模式，仅解析文件头不初始化完整解码器。需后续调用 ov_test_open() 完成初始化。适用于边下载边检测音频格式的场景。

#### ov_open1() / ov_open2()

```c
int ov_open1(OggVorbis_File *vf);
int ov_open2(OggVorbis_File *vf);
```

分步初始化接口。ov_open1() 解析文件头，ov_open2() 完成解码器初始化。用于需要细粒度控制初始化过程的场景。

#### ov_clear()

```c
int ov_clear(OggVorbis_File *vf);
```

关闭已打开的 Vorbis 文件，释放所有相关资源。使用完文件后必须调用此函数。

### 音频解码

#### ov_read()

```c
long ov_read(OggVorbis_File *vf, char *buffer, int length,
             int bigendianp, int word, int sgned, int *bitstream);
```

解码并返回下一帧 PCM 数据。参数说明：
- buffer：输出缓冲区
- length：缓冲区长度（字节）
- bigendianp：字节序（0=小端）
- word：采样宽度（1=8bit, 2=16bit, 3=24bit, 4=32bit）
- sgned：是否有符号（0=无符号，1=有符号）
- bitstream：输出当前 bitstream 索引

返回读取的字节数，0 表示文件结束，负值表示错误。

#### ov_read_float()

```c
long ov_read_float(OggVorbis_File *vf, float ***pcm_channels, int samples,
                   int *bitstream);
```

解码并返回浮点格式的 PCM 数据。pcm_channels 是指向多声道浮点缓冲区的指针数组，samples 指定每声道采样数。返回实际读取的采样数。

### 文件信息查询

#### ov_pcm_total() / ov_time_total()

```c
ogg_int64_t ov_pcm_total(OggVorbis_File *vf, int i);
double ov_time_total(OggVorbis_File *vf, int i);
```

获取音频总时长。i 为逻辑流索引（-1 表示当前流）。ov_pcm_total() 返回 PCM 采样总数，ov_time_total() 返回秒为单位的时长。

#### ov_info()

```c
vorbis_info *ov_info(OggVorbis_File *vf, int link);
```

获取指定流的 vorbis_info 信息，包含采样率、声道数和码率等。

#### ov_comment()

```c
vorbis_comment *ov_comment(OggVorbis_File *vf, int link);
```

获取指定流的 vorbis_comment 信息，包含艺术家、专辑等元数据。

### 跳转/Seek

#### ov_pcm_seek()

```c
int ov_pcm_seek(OggVorbis_File *vf, ogg_int64_t pos);
```

跳转到指定 PCM 采样位置。pos 是从文件开始的 PCM 采样偏移。

#### ov_time_seek()

```c
int ov_time_seek(OggVorbis_File *vf, double pos);
```

跳转到指定时间位置。pos 是以秒为单位的播放时间。

### 低级合成 API

以下为 vorbis 合成层的底层 API：

#### vorbis_synthesis_init()

```c
int vorbis_synthesis_init(vorbis_dsp_state *v, vorbis_info *vi);
```

初始化 DSP 合成器，将 vorbis_info 配置应用到 dsp_state。

#### vorbis_block_init()

```c
int vorbis_block_init(vorbis_dsp_state *v, vorbis_block *vb);
```

初始化 vorbis_block 音频块，关联到指定的 dsp_state。

#### vorbis_synthesis()

```c
int vorbis_synthesis(vorbis_block *vb, ogg_packet *op);
```

将一个 OGG 数据包解码为 PCM 样本到 block 中。解码后的 PCM 通过 vorbis_synthesis_pcmout() 提取。

## 错误码

Vorbis API 返回负值表示错误：
- `OV_FALSE`：通用失败
- `OV_EOF`：文件结束
- `OV_HOLE`：数据丢失
- `OV_EREAD`：读取错误
- `OV_EFAULT`：内部错误
- `OV_EINVAL`：无效参数
- `OV_ENOTVORBIS`：非 Vorbis 文件
- `OV_EBADHEADER`：损坏的文件头
- `OV_EVERSION`：不支持的 Vorbis 版本
- `OV_ENOSEEK`：流不可跳转

## 代码示例

以下示例展示如何使用 Vorbis API 打开 OGG 文件并解码播放：

```c
#include <vorbis/vorbisfile.h>
#include <stdio.h>

#define PCM_BUFFER_SIZE 4096

int play_vorbis_file(const char *filename)
{
    OggVorbis_File vf;
    FILE *fp;
    char buffer[PCM_BUFFER_SIZE];
    int  bitstream;
    long bytes_read;
    vorbis_info *vi;

    /* 打开文件 */
    if (ov_fopen(filename, &vf) != 0) {
        printf("Failed to open file: %s\n", filename);
        return -1;
    }

    /* 获取音频流信息 */
    vi = ov_info(&vf, -1);
    printf("Channels: %d, Rate: %ld Hz\n", vi->channels, vi->rate);

    /* 循环读取并解码 */
    while ((bytes_read = ov_read(&vf, buffer, sizeof(buffer),
                                  0, 2, 1, &bitstream)) > 0) {
        /* 将 PCM 数据送入音频输出
         * 此处可调用平台相关的音频播放 API
         */
        // audio_write(buffer, bytes_read);
    }

    /* 关闭文件 */
    ov_clear(&vf);
    return 0;
}
```

使用浮点格式解码的示例：

```c
int play_vorbis_float(const char *filename)
{
    OggVorbis_File vf;
    float **pcm_channels;
    long samples;
    int  bitstream, i, ch;

    if (ov_fopen(filename, &vf) != 0) {
        return -1;
    }

    while ((samples = ov_read_float(&vf, &pcm_channels, 4096, &bitstream)) > 0) {
        vorbis_info *vi = ov_info(&vf, -1);
        int channels = vi->channels;

        /* 处理多声道数据
         * pcm_channels[0] 是左声道
         * pcm_channels[1] 是右声道
         * 以此类推
         */
        for (ch = 0; ch < channels; ch++) {
            for (i = 0; i < samples; i++) {
                float sample = pcm_channels[ch][i];
                /* 处理 sample... */
            }
        }
    }

    ov_clear(&vf);
    return 0;
}
```

## 参考

- Xiph.Org Foundation: https://xiph.org/vorbis/
- Vorbis 官方文档: https://xiph.org/vorbis/doc/
- OGG 容器格式规范: https://xiph.org/ogg/doc/
