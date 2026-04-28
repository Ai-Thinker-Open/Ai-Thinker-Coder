# AMR 语音编解码器

## 概述

AMR (Adaptive Multi-Rate) 是 3GPP 标准定义的语音编解码器系列，广泛应用于移动通信领域的语音通话场景。AMR 系列包括两种主要规格：

| 类型 | 采样率 | 比特率范围 | 帧时长 | 应用场景 |
|------|--------|------------|--------|----------|
| AMR-NB (窄带) | 8 kHz | 4.75 ~ 12.2 kbps | 20 ms | 传统移动语音通话 |
| AMR-WB (宽带) | 16 kHz | 6.6 ~ 23.85 kbps | 20 ms | VoLTE、高清语音通话 |

BL618 芯片内置 AMR 硬件加速器，支持 AMR-NB 和 AMR-WB 解码功能，适用于 IP 语音电话、无线对讲机等产品开发。

---

## AMR-NB 解码器

AMR-NB (Adaptive Multi-Rate Narrowband) 是针对 8 kHz 采样率设计的语音编解码器，最初用于 GSM 网络，后被 3GPP 标准采纳。

### 帧类型

AMR-NB 支持 8 种编码模式（比特率）：

| 模式 | 比特率 | 每帧比特数 | 用途 |
|------|--------|------------|------|
| 0 | 4.75 kbps | 95 | 最低码率 |
| 1 | 5.15 kbps | 103 | - |
| 2 | 5.90 kbps | 118 | - |
| 3 | 6.70 kbps | 134 | 默认推荐 |
| 4 | 7.40 kbps | 148 | - |
| 5 | 7.95 kbps | 159 | - |
| 6 | 10.2 kbps | 204 | 最高码率(ETSI) |
| 7 | 12.2 kbps | 244 | 最高码率(AMR) |

此外包含 SID (Silence Descriptor) 帧用于传输背景噪声信息。

### 数据结构

AMR-NB 解码器使用 `void *` 类型的内部状态指针，开发者需先查询内存需求，分配足够缓冲区后再初始化。

### API 函数

#### Memory_Associated_AMRNB_Decoder

获取 AMR-NB 解码器所需的内存大小。

```c
Word32 Memory_Associated_AMRNB_Decoder(void);
```

**返回值：** 解码器所需的字节数

**示例：**
```c
Word32 mem_size = Memory_Associated_AMRNB_Decoder();
printf("AMR-NB decoder requires %lu bytes\r\n", mem_size);
```

#### Decoder_Index_AMRNB_Interface_init

初始化 AMR-NB 解码器实例。

```c
void Decoder_Index_AMRNB_Interface_init(void *state_data);
```

**参数：**
- `state_data` - 指向已分配内存区域的指针，用于存储解码器内部状态

**示例：**
```c
void *amrnb_state = malloc(mem_size);
if (amrnb_state == NULL) {
    // 内存分配失败
    return -1;
}
Decoder_Index_AMRNB_Interface_init(amrnb_state);
```

#### Decoder_Index_AMRNB_Interface

解码一帧 AMR-NB 语音数据。

```c
Word16 Decoder_Index_AMRNB_Interface(
    void           *state_data,       // 解码器状态指针
    enum Frame_Type_3GPP frame_type, // 帧类型 (0-8)
    UWord8         *speech_bits_ptr,  // 输入比特流
    Word16         *raw_pcm_buffer,    // 输出 PCM 缓冲区
    bitstream_format input_format     // 输入格式 (WMF/IF2/ETS)
);
```

**参数：**
- `state_data` - 解码器状态指针（由 init 初始化）
- `frame_type` - AMR-NB 帧类型，枚举值 `AMR_475, AMR_515, AMR_59, AMR_67, AMR_74, AMR_795, AMR_102, AMR_122, AMR_SID`
- `speech_bits_ptr` - 指向压缩比特流的指针
- `raw_pcm_buffer` - 指向输出 PCM 缓冲区的指针（需至少 160 个 Word16）
- `input_format` - 输入比特流格式：`WMF`(Wireless MSF), `IF2`(Interface Format 2), `ETS`(ETS 300 725)

**返回值：** 0 表示成功

**输出 PCM：** 每帧输出 160 个采样点（20ms @ 8kHz）

#### Decoder_Index_AMRNB_Interface_exit

释放 AMR-NB 解码器资源。

```c
void Decoder_Index_AMRNB_Interface_exit(void *state_data);
```

**参数：**
- `state_data` - 解码器状态指针

**示例：**
```c
Decoder_Index_AMRNB_Interface_exit(amrnb_state);
free(amrnb_state);
amrnb_state = NULL;
```

---

## AMR-WB 解码器

AMR-WB (Adaptive Multi-Rate Wideband) 是针对 16 kHz 采样率设计的宽带语音编解码器，提供更自然的语音质量和更好的可懂度。

### 帧类型

AMR-WB 支持 9 种编码模式：

| 模式 | 比特率 | 每帧比特数 | 用途 |
|------|--------|------------|------|
| 0 | 6.60 kbps | 132 | 最低码率 |
| 1 | 8.85 kbps | 177 | - |
| 2 | 12.65 kbps | 253 | - |
| 3 | 14.25 kbps | 285 | - |
| 4 | 15.85 kbps | 317 | - |
| 5 | 18.25 kbps | 365 | - |
| 6 | 19.85 kbps | 397 | - |
| 7 | 23.05 kbps | 461 | - |
| 8 | 23.85 kbps | 477 | 最高码率 |

同样包含 SID 帧用于传输舒适噪声参数。

### API 函数

#### AMR_WB_get_memreq

获取 AMR-WB 解码器所需的内存大小。

```c
Word32 AMR_WB_get_memreq(void);
```

**返回值：** 解码器所需的字节数

#### AMR_WB_dec_init

初始化 AMR-WB 解码器实例。

```c
int AMR_WB_dec_init(void **spd_state, void *st, int16 **scratch_mem);
```

**参数：**
- `spd_state` - 输出参数，返回解码器状态指针
- `st` - 已分配的内存缓冲区
- `scratch_mem` - 输出参数，返回临时工作内存指针

**返回值：** 0 表示成功

#### AMR_WB_decode

解码一帧 AMR-WB 语音数据。

```c
int AMR_WB_decode(
    int16  mode,           // 编码模式 (0-8)
    int16  prms[],         // 输入参数向量
    int16  synth16k[],     // 输出合成语音
    int16 *frame_length,   // 输出帧长度
    void  *spd_state,      // 解码器状态指针
    int16  frame_type,     // 帧类型
    int16  scratch_mem[]   // 临时工作内存
);
```

**参数：**
- `mode` - AMR-WB 编码模式 (0-8)
- `prms` - 指向压缩比特流参数的指针
- `synth16k` - 输出缓冲区，存储 16kHz 采样的 PCM 语音（需至少 320 个 int16）
- `frame_length` - 输出参数，实际输出的采样点数
- `spd_state` - 解码器状态指针
- `frame_type` - AMR-WB 帧类型
- `scratch_mem` - 临时工作内存

**返回值：** 0 表示成功

**输出 PCM：** 每帧输出 320 个采样点（20ms @ 16kHz）

#### AMR_WB_dec_deinit

释放 AMR-WB 解码器资源。

```c
void AMR_WB_dec_deinit(void *spd_state);
```

**参数：**
- `spd_state` - 解码器状态指针

---

## 使用流程

### AMR-NB 解码完整流程

```c
#include "amrdecode.h"
#include "frame_type_3gpp.h"

// 1. 获取内存需求
Word32 mem_size = Memory_Associated_AMRNB_Decoder();

// 2. 分配内存
void *amrnb_state = malloc(mem_size);
if (amrnb_state == NULL) {
    return -1;
}

// 3. 初始化解码器
Decoder_Index_AMRNB_Interface_init(amrnb_state);

// 4. 分配输入输出缓冲区
UWord8 speech_bits[32];        // 输入比特流
Word16 pcm_buffer[160];       // 输出 PCM (160 samples @ 8kHz)

// 5. 循环解码每帧
enum Frame_Type_3GPP frame_type = AMR_67;  // 假设使用 6.7kbps 模式
for (int i = 0; i < frame_count; i++) {
    // 填充压缩比特流到 speech_bits
    // ...

    // 解码一帧
    Decoder_Index_AMRNB_Interface(
        amrnb_state,
        frame_type,
        speech_bits,
        pcm_buffer,
        IF2  // Interface Format 2
    );

    // 处理 PCM 数据 (pcm_buffer 包含 160 个 16-bit 样本)
    // ...
}

// 6. 销毁解码器
Decoder_Index_AMRNB_Interface_exit(amrnb_state);
free(amrnb_state);
```

### AMR-WB 解码完整流程

```c
#include "pvamrwbdecoder.h"

// 1. 获取内存需求
Word32 mem_size = AMR_WB_get_memreq();

// 2. 分配内存
void *amrwb_buf = malloc(mem_size);
void *amrwb_state = NULL;
int16 *scratch_mem = NULL;

// 3. 初始化解码器
AMR_WB_dec_init(&amrwb_state, amrwb_buf, &scratch_mem);

// 4. 分配输入输出缓冲区
int16 prms[64];                // 输入参数
int16 synth16k[320];           // 输出 PCM (320 samples @ 16kHz)
int16 frame_length = 0;

// 5. 循环解码每帧
for (int i = 0; i < frame_count; i++) {
    // 填充压缩参数到 prms
    // ...

    // 解码一帧
    AMR_WB_decode(
        2,                      // mode 2 (12.65 kbps)
        prms,
        synth16k,
        &frame_length,
        amrwb_state,
        0,                      // frame_type
        scratch_mem
    );

    // 处理 PCM 数据 (synth16k 包含 frame_length 个 16-bit 样本)
    // ...
}

// 6. 销毁解码器
AMR_WB_dec_deinit(amrwb_state);
free(amrwb_buf);
```

---

## 采样率与帧长

| 编解码器 | 采样率 | 每帧采样数 | 帧时长 |
|----------|--------|------------|--------|
| AMR-NB | 8 kHz | 160 | 20 ms |
| AMR-WB | 16 kHz | 320 | 20 ms |

**计算示例：**
- AMR-NB: 8000 Hz × 0.02 s = 160 samples
- AMR-WB: 16000 Hz × 0.02 s = 320 samples

---

## 输入格式说明

AMR 比特流支持多种封装格式：

| 格式 | 说明 | 适用场景 |
|------|------|----------|
| WMF | Wireless MSF 格式 | 早期 GSM 实现 |
| IF2 | Interface Format 2 | 3GPP 标准推荐 |
| ETS | ETS 300 725 格式 | 欧洲电信标准 |

BL618 SDK 推荐使用 IF2 格式，该格式与 3GPP TS 26.101 规范一致。

---

## 注意事项

1. **内存对齐**：分配的解码器状态缓冲区建议按 4 字节对齐，以确保 ARM 架构上的访问效率。

2. **帧类型同步**：解码前需正确识别输入帧的类型（比特率模式），错误的帧类型会导致解码失败或输出杂音。

3. **PCM 输出范围**：解码输出的 PCM 数据为 16-bit 有符号整型，范围 [-32768, 32767]，需进行适当的音量调整后再送入 DAC。

4. **边界情况**：对于破损或丢失的帧，建议使用上一帧的解码结果进行舒适噪声填充，以改善听觉体验。

5. **线程安全**：解码器实例不可跨线程共享，每个并发解码通道需独立分配解码器状态。

---

## 参考

- 3GPP TS 26.073: ANSI-C code for the Adaptive Multi-Rate (AMR) speech codec
- 3GPP TS 26.173: ANSI-C code for the Adaptive Multi-Rate - Wideband (AMR-WB) speech codec
- 3GPP TS 26.101: Frame structure for Adaptive Multi-Rate - Wideband (AMR-WB) speech codec
