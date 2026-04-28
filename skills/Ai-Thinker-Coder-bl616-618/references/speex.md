# Speex 语音编解码器

## 概述

Speex 是 Xiph.Org 基金会开发的开源语音编解码器，基于 CELP（码激励线性预测）算法，专为 VoIP（Voice over IP）应用设计。与其他商业编解码器不同，Speex 采用 BSD 许可证，可免费用于商业和非商业项目。

Speex 支持三种采样率模式：

| 模式 | 采样率 | 典型应用场景 |
|------|--------|--------------|
| 窄带（Narrowband） | 8 kHz | 传统电话、语音通话 |
| 宽带（Wideband） | 16 kHz | 视频会议、IP 语音 |
| 超宽带（Ultra-Wideband） | 32 kHz | 高质量语音传输 |

BL618 芯片集成了 Speex 编解码库，可用于开发 VoIP 电话、语音聊天、语音记录等多媒体应用。

## 关键数据类型

### SpeexBits

`SpeexBits` 结构体用于管理比特流的打包与解包：

```c
typedef struct SpeexBits {
   char *chars;      // 原始数据缓冲区
   int   nbBits;     // 流中存储的总比特数
   int   charPtr;    // 字节"光标"位置
   int   bitPtr;     // 当前字节内的比特"光标"位置
   int   owner;      // 结构体是否"拥有"原始缓冲区
   int   overflow;  // 尝试读取超出有效数据时置 1
   int   buf_size;   // 缓冲区分配大小
} SpeexBits;
```

### SpeexMode

`SpeexMode` 结构体定义了一种编解码模式，包括编码器/解码器的初始化、编码、解码及控制函数指针：

```c
typedef struct SpeexMode {
   const void       *mode;          // 低级模式数据指针
   mode_query_func   query;         // 模式查询函数
   const char       *modeName;      // 模式名称
   int               modeID;        // 模式 ID
   encoder_init_func enc_init;       // 编码器初始化函数
   encoder_destroy_func enc_destroy; // 编码器销毁函数
   encode_func       enc;           // 帧编码函数
   decoder_init_func dec_init;      // 解码器初始化函数
   decoder_destroy_func dec_destroy; // 解码器销毁函数
   decode_func       dec;           // 帧解码函数
   encoder_ctl_func  enc_ctl;       // 编码器控制函数
   decoder_ctl_func  dec_ctl;       // 解码器控制函数
} SpeexMode;
```

### SpeexEncoderState 和 SpeexDecoderState

编码器状态和解码器状态为不透明指针类型，由库内部维护。用户通过这些指针操作编解码器，但不应直接访问其内部结构：

```c
void *speex_encoder_init(const SpeexMode *mode);
void *speex_decoder_init(const SpeexMode *mode);
```

## 核心 API

### 编解码器初始化与销毁

```c
// 初始化窄带编码器
void *speex_encoder_init(const SpeexMode *mode);

// 初始化解码器
void *speex_decoder_init(const SpeexMode *mode);

// 销毁编码器，释放资源
void speex_encoder_destroy(void *state);

// 销毁解码器，释放资源
void speex_decoder_destroy(void *state);
```

### 编码与解码

```c
// 编码一帧语音（浮点输入）
// state: 编码器状态
// in: 输入 PCM 数据，范围 ±2^15
// bits: 比特流输出
// 返回值: 0 表示帧无需传输（仅 DTX 模式），1 表示正常
int speex_encode(void *state, float *in, SpeexBits *bits);

// 编码一帧语音（整数输入）
int speex_encode_int(void *state, spx_int16_t *in, SpeexBits *bits);

// 解码一帧语音（浮点输出）
// bits: 比特流输入（若为 NULL 表示丢包）
// out: 解码输出 PCM 数据
// 返回值: 0 正常，-1 流结束，-2 流损坏
int speex_decode(void *state, SpeexBits *bits, float *out);

// 解码一帧语音（整数输出）
int speex_decode_int(void *state, SpeexBits *bits, spx_int16_t *out);
```

### 比特流操作

```c
// 初始化 SpeexBits 结构体
void speex_bits_init(SpeexBits *bits);

// 使用预分配缓冲区初始化
void speex_bits_init_buffer(SpeexBits *bits, void *buff, int buf_size);

// 从内存区域读取数据初始化比特流
void speex_bits_read_from(SpeexBits *bits, const char *bytes, int len);

// 将比特流内容写入内存
// 返回写入的字节数
int speex_bits_write(SpeexBits *bits, char *bytes, int max_len);

// 销毁 SpeexBits，释放资源
void speex_bits_destroy(SpeexBits *bits);

// 重置比特流到初始状态
void speex_bits_reset(SpeexBits *bits);
```

### 模式获取

```c
// 获取指定模式的 SpeexMode 指针
// mode: SPEEX_MODEID_NB (0), SPEEX_MODEID_WB (1), SPEEX_MODEID_UWB (2)
const SpeexMode *speex_lib_get_mode(int mode);
```

预定义模式常量：

```c
extern const SpeexMode speex_nb_mode;   // 窄带模式
extern const SpeexMode speex_wb_mode;   // 宽带模式
extern const SpeexMode speex_uwb_mode;  // 超宽带模式
```

### 参数控制

`speex_encoder_ctl` 和 `speex_decoder_ctl` 用于设置/获取编解码器参数，用法类似 ioctl：

```c
int speex_encoder_ctl(void *state, int request, void *ptr);
int speex_decoder_ctl(void *state, int request, void *ptr);
```

常用控制命令：

| 控制命令 | 说明 | 值范围 |
|---------|------|--------|
| `SPEEX_SET_QUALITY` | 设置编码质量 | 0~10 |
| `SPEEX_GET_FRAME_SIZE` | 获取帧大小 | - |
| `SPEEX_SET_COMPLEXITY` | 设置编码复杂度 | 0~10 |
| `SPEEX_SET_VBR` | 设置变比特率 | 0 或 1 |
| `SPEEX_SET_VBR_QUALITY` | 设置 VBR 质量 | 0~10 |
| `SPEEX_SET_ABR` | 设置平均比特率 | bps |
| `SPEEX_SET_DTX` | 设置非连续传输 | 0 或 1 |
| `SPEEX_SET_SAMPLING_RATE` | 设置采样率 | Hz |
| `SPEEX_GET_BITRATE` | 获取当前比特率 | - |
| `SPEEX_SET_ENH` | 设置增强（解码器） | 0 或 1 |
| `SPEEX_SET_HIGHPASS` | 设置高通滤波 | 0 或 1 |

## 质量等级

Speex 支持 0~10 共 11 个质量等级，数值越高编码质量越好，但比特率也越高：

| 质量等级 | 窄带比特率 | 宽带比特率 | 典型用途 |
|---------|-----------|-----------|---------|
| 0 | ~2.15 kbps | ~3.95 kbps | 极低带宽 |
| 1 | ~5.9 kbps | ~7.75 kbps | 很低带宽 |
| 2-3 | 8~11 kbps | 9~12 kbps | 低带宽 |
| 4-6 | 11~18 kbps | 12~22 kbps | 平衡质量/带宽 |
| 7-8 | 18~24 kbps | 22~28 kbps | 较高质量 |
| 9-10 | 24~44 kbps | 28~44 kbps | 最高质量 |

宽带模式下，相同质量等级通常比窄带模式占用更多比特率。

## 使用流程

典型 Speex 编码/解码流程：

```
1. speex_bits_init()        - 初始化比特流结构
2. speex_lib_get_mode()      - 获取编解码模式
3. speex_encoder_init()      - 初始化编码器
4. speex_decoder_init()      - 初始化解码器
5. [可选] speex_encoder_ctl() - 设置质量等参数
6. 循环:
   a. speex_encode()         - 编码一帧
   b. speex_bits_write()     - 写入网络缓冲区
   c. speex_bits_read_from() - 从网络读取
   d. speex_decode()         - 解码
7. speex_encoder_destroy()   - 销毁编码器
8. speex_decoder_destroy()   - 销毁解码器
9. speex_bits_destroy()      - 销毁比特流
```

## 代码示例

以下示例展示在 BL618 上使用 Speex 进行语音编解码的基本流程：

```c
#include "speex/speex.h"
#include "speex/speex_bits.h"

#define FRAME_SIZE 160  // 窄带 8kHz/16bit = 20ms 帧

void speex_voip_example(void)
{
    // 比特流结构
    SpeexBits enc_bits;
    SpeexBits dec_bits;
    SpeexBits *bits = &enc_bits;

    // PCM 数据缓冲区
    spx_int16_t pcm_in[FRAME_SIZE];
    spx_int16_t pcm_out[FRAME_SIZE];
    char encoded[256];

    // 编码器和解码器状态
    void *enc_state;
    void *dec_state;

    const SpeexMode *mode;

    // 步骤 1: 初始化比特流
    speex_bits_init(&enc_bits);
    speex_bits_init(&dec_bits);

    // 步骤 2: 获取窄带模式
    mode = speex_lib_get_mode(SPEEX_MODEID_NB);

    // 步骤 3: 初始化编码器和解码器
    enc_state = speex_encoder_init(mode);
    dec_state = speex_decoder_init(mode);

    // 步骤 4: 设置编码质量 (0-10)
    int quality = 4;
    speex_encoder_ctl(enc_state, SPEEX_SET_QUALITY, &quality);

    // 步骤 5: 设置变比特率（可选）
    int vbr = 1;
    speex_encoder_ctl(enc_state, SPEEX_SET_VBR, &vbr);

    // 获取帧大小
    int frame_size;
    speex_encoder_ctl(enc_state, SPEEX_GET_FRAME_SIZE, &frame_size);

    // 编码示例
    // 假设 pcm_in 已填充原始 PCM 数据
    speex_bits_reset(bits);                    // 重置比特流
    speex_encode_int(enc_state, pcm_in, bits);  // 编码
    int nb_bytes = speex_bits_write(bits, encoded, sizeof(encoded));

    // 解码示例
    speex_bits_read_from(&dec_bits, encoded, nb_bytes);
    int ret = speex_decode_int(dec_state, &dec_bits, pcm_out);

    // 步骤 6: 销毁资源
    speex_encoder_destroy(enc_state);
    speex_decoder_destroy(dec_state);
    speex_bits_destroy(&enc_bits);
    speex_bits_destroy(&dec_bits);
}
```

### 宽带模式示例

```c
void speex_wideband_example(void)
{
    SpeexBits bits;
    void *enc_state, *dec_state;
    spx_int16_t pcm_in[320];  // 宽带帧更大 (16kHz * 20ms)
    spx_int16_t pcm_out[320];

    speex_bits_init(&bits);

    // 使用宽带模式
    const SpeexMode *mode = speex_lib_get_mode(SPEEX_MODEID_WB);
    enc_state = speex_encoder_init(mode);
    dec_state = speex_decoder_init(mode);

    int quality = 6;
    speex_encoder_ctl(enc_state, SPEEX_SET_QUALITY, &quality);

    // 编码
    speex_bits_reset(&bits);
    speex_encode_int(enc_state, pcm_in, &bits);
    char encoded[256];
    int nb_bytes = speex_bits_write(&bits, encoded, sizeof(encoded));

    // 解码
    speex_bits_read_from(&bits, encoded, nb_bytes);
    speex_decode_int(dec_state, &bits, pcm_out);

    // 销毁
    speex_encoder_destroy(enc_state);
    speex_decoder_destroy(dec_state);
    speex_bits_destroy(&bits);
}
```

## 丢包补偿（PLC）

Speex 内置丢包补偿（Packet Loss Concealment）功能。当网络丢包时，解码器可以将 `bits` 参数设为 `NULL` 调用 `speex_decode()`，此时使用上一帧数据外推生成补偿音频：

```c
// 丢包时解码
speex_decode_int(dec_state, NULL, pcm_out);  // bits 为 NULL
```

丢包补偿会使用静音或上一帧的某种变形来替代丢失的语音，虽然质量不如正常解码，但能避免解码器产生刺耳噪声。

## 注意事项

1. **帧大小匹配**：编码器和解码器必须使用相同的帧大小。调用 `SPEEX_GET_FRAME_SIZE` 获取当前模式的帧大小。

2. **字节序**：Speex 编码的数据是字节序相关的，在不同平台间传输时需要注意。

3. **内存管理**：`SpeexBits` 结构体中的 `chars` 缓冲区由 `speex_bits_init()` 分配，`speex_bits_destroy()` 会释放。若使用 `speex_bits_init_buffer()` 初始化，则外部管理缓冲区生命周期。

4. **VBR 质量设置**：启用变比特率（VBR）后，设置 `SPEEX_SET_VBR_QUALITY` 来控制目标质量，编码器会自动在保证质量和限制比特率之间平衡。

5. **DTX 非连续传输**：启用 DTX 后，编码器在检测到静音时会停止发送数据，可节省带宽约 50%。

## 参考

- Xiph.Org Foundation: https://www.xiph.org/
- Speex 官方文档: https://www.speex.org/docs/
- Bouffalo SDK Speex 组件源码: `components/multimedia/speex/include/speex/`
