# PVMP4AudioDecoder AAC 软件解码器

## 概述

PVMP4AudioDecoder 是 PacketVideo 开发的开源 AAC-LC/AAC+/eAAC+ 软件解码器，已集成到 Bouffalo SDK 的 `multimedia/aacdec` 组件中。该解码器完全基于 ANSI C 实现，适用于嵌入式系统音频播放场景。

BL618 通过该解码器组件实现 AAC 音频格式的软解，支持以下音频类型：

| 音频类型 | 说明 | 采样率范围 |
|---------|------|-----------|
| AAC-LC | 低复杂度 AAC | 8-96 kHz |
| AACPlus | AAC+ (带 SBR) | 24-48 kHz 输入，输出 2x 上采样 |
| enhanced AACPlus | 增强型 AAC+ (带 SBR + PS) | 同上 |

## 头文件

```c
#include "pvmp4audiodecoder_api.h"
#include "pv_audio_type_defs.h"
```

## 关键数据类型

### tPVMP4AudioDecoderExternal

解码器外部控制结构，用于与解码库交换输入输出数据：

```c
typedef struct tPVMP4AudioDecoderExternal
{
    /* 输入字段 */
    UChar  *pInputBuffer;              // AAC 码流输入缓冲区
    Int     inputBufferCurrentLength; // 输入缓冲区有效字节数
    Int     inputBufferMaxLength;      // 输入缓冲区总大小
    Int     desiredChannels;           // 期望输出声道数 (1=单声道, 2=立体声)

    /* 输入/输出字段 */
    Int     inputBufferUsedLength;     // 解码消耗的字节数
    Int32   remainderBits;            // 跨帧剩余比特数

    /* 输出字段 */
    Int32   samplingRate;             // 采样率 (samples/sec)
    Int32   bitRate;                  // 码率 (bits/sec)
    Int     encodedChannels;          // 码流原始声道数
    Int     frameLength;              // 输出 PCM 样本数/声道 (固定 1024)
    Int     audioObjectType;          // 音频对象类型
    Int     extendedAudioObjectType;  // 扩展音频对象类型

    /* AAC+ 相关 */
    Int16  *pOutputBuffer;            // PCM 主输出缓冲区 (2048 样本)
    Int16  *pOutputBuffer_plus;       // AAC+ 输出缓冲区
    Int32   aacPlusUpsamplingFactor; // 上采样因子 (通常为 2)
    Bool    aacPlusEnabled;           // AAC+ 使能标志

    /* 控制标志 */
    Bool    repositionFlag;           // 跳转标记 (快进/快退时设置)
    tPVMP4AudioDecoderOutputFormat outputFormat; // 输出格式
} tPVMP4AudioDecoderExternal;
```

### AACDecStatus (解码状态码)

```c
typedef enum ePVMP4AudioDecoderErrorCode
{
    AAC_SUCCESS           =  0,  // 解码成功
    AAC_INVALID_FRAME     = 10,  // 无效帧
    AAC_INCOMPLETE_FRAME  = 20,  // 输入数据不完整
    AAC_LOST_FRAME_SYNC   = 30   // 帧同步丢失
} AACDecStatus;
```

### STREAMTYPE (码流类型)

```c
typedef enum
{
    AAC       = 0,   // 普通 AAC-LC
    AACPLUS   = 1,   // AAC+ (SBR)
    ENH_AACPLUS = 2  // 增强 AAC+ (SBR + PS)
} STREAMTYPE;
```

### 输出格式枚举

```c
typedef enum ePVMP4AudioDecoderOutputFormat
{
    OUTPUTFORMAT_16PCM_GROUPED     = 0,  // 分组格式: LLLL...RRRR
    OUTPUTFORMAT_16PCM_INTERLEAVED = 1   // 交错格式: LRLRLR...
} tPVMP4AudioDecoderOutputFormat;
```

## 内存需求

### AACDecMemReq()

获取解码器所需内存大小。在初始化解码器前，必须先调用此函数分配足够内存。

```c
UInt32 AACDecMemReq(void);
```

**返回值:** 解码器所需内存字节数

**示例:**
```c
UInt32 memSize = AACDecMemReq();
void *pDecoderMem = malloc(memSize);
if (pDecoderMem == NULL) {
    // 内存分配失败
}
```

## 核心 API

### AACInitDecoder() - 初始化解码器

```c
Int AACInitDecoder(
    tPVMP4AudioDecoderExternal *pExt,  // 解码器外部结构指针
    void                       *pMem  // 解码器工作内存
);
```

**参数说明:**
- `pExt`: 指向 `tPVMP4AudioDecoderExternal` 结构体的指针
- `pMem`: 指向通过 `AACDecMemReq()` 获取的内存块

**返回值:** `AAC_SUCCESS` (0) 表示成功

**初始化步骤:**
1. 调用 `AACDecMemReq()` 获取所需内存大小
2. 分配内存块
3. 初始化 `tPVMP4AudioDecoderExternal` 结构体
4. 调用 `AACInitDecoder()` 完成初始化

---

### AACDecodeFrame() - 解码一帧

```c
Int AACDecodeFrame(
    tPVMP4AudioDecoderExternal *pExt,  // 解码器外部结构
    void                       *pMem  // 解码器工作内存
);
```

**输入要求:**
- `pInputBuffer`: 指向包含一帧 AAC 数据的缓冲区
- `inputBufferCurrentLength`: 输入数据字节数
- `pOutputBuffer`: 指向至少 2048 样本的输出缓冲区
- `desiredChannels`: 期望输出声道数

**输出结果:**
- `inputBufferUsedLength`: 实际消耗的输入字节数
- `samplingRate`: 解码得到的采样率
- `frameLength`: 输出 PCM 样本数 (通常为 1024)
- `pOutputBuffer`: 16-bit PCM 数据

**返回值:**
- `AAC_SUCCESS`: 解码成功
- `AAC_INVALID_FRAME`: 无效帧
- `AAC_INCOMPLETE_FRAME`: 输入数据不足

---

### AACDecodeAudioSpecificConfig() - 解析 ASC

解析 Audio Specific Config (ASC) 配置信息，用于从码流中提取音频参数。

```c
Int AACDecodeAudioSpecificConfig(
    tPVMP4AudioDecoderExternal *pExt,  // 解码器外部结构
    void                       *pMem  // 解码器工作内存
);
```

**典型应用场景:**
- 从 ADTS 头部解析采样率和声道配置
- 手动设置音频参数时校验配置

---

### AACFreeDecoder() - 释放解码器

由于解码器采用纯软件实现，不包含独立的释放函数。释放内存只需直接 `free()` 之前分配的内存块：

```c
free(pDecoderMem);
pDecoderMem = NULL;
```

**注意:** 释放前应确保没有正在进行的解码操作。

## 输出格式

解码器输出 **PCM 16-bit interleaved** 立体声数据：

```
采样点序列: L0 R0 L1 R1 L2 R2 ... L1023 R1023
字节序:     小端序 (Little Endian)
声道格式:   交错 (Interleaved)
样本范围:   -32768 ~ 32767
```

对于单声道输出，缓冲区仅包含左声道数据。

**输出缓冲区大小计算:**
```c
// 立体声: 1024 样本/声道 × 2 声道 × 2 字节 = 4096 字节
// 单声道: 1024 样本/声道 × 1 声道 × 2 字节 = 2048 字节
```

## 代码示例

### 完整解码流程

```c
#include "pvmp4audiodecoder_api.h"
#include "pv_audio_type_defs.h"
#include <stdlib.h>
#include <stdio.h>

int aac_decode_example(const uint8_t *aac_data, uint32_t aac_len,
                       int16_t *pcm_out, uint32_t *pcm_samples)
{
    tPVMP4AudioDecoderExternal decExt = {0};
    UInt32 memSize;
    void *pDecoderMem;
    Int ret;

    /* 步骤1: 获取内存需求 */
    memSize = AACDecMemReq();

    /* 步骤2: 分配解码器内存 */
    pDecoderMem = malloc(memSize);
    if (pDecoderMem == NULL) {
        printf("Decoder memory allocation failed\r\n");
        return -1;
    }

    /* 步骤3: 初始化外部结构 */
    decExt.pInputBuffer          = (UChar *)aac_data;
    decExt.inputBufferCurrentLength = aac_len;
    decExt.inputBufferMaxLength  = aac_len;
    decExt.pOutputBuffer         = pcm_out;
    decExt.pOutputBuffer_plus    = NULL;
    decExt.desiredChannels       = 2;  // 输出立体声
    decExt.outputFormat          = OUTPUTFORMAT_16PCM_INTERLEAVED;
    decExt.aacPlusEnabled        = TRUE;
    decExt.aacPlusUpsamplingFactor = 2;
    decExt.repositionFlag        = FALSE;
    decExt.inputBufferUsedLength = 0;
    decExt.remainderBits         = 0;

    /* 步骤4: 初始化解码器 */
    ret = AACInitDecoder(&decExt, pDecoderMem);
    if (ret != AAC_SUCCESS) {
        printf("Decoder init failed: %d\r\n", ret);
        free(pDecoderMem);
        return -1;
    }

    /* 步骤5: 解码一帧 */
    ret = AACDecodeFrame(&decExt, pDecoderMem);
    if (ret == AAC_SUCCESS) {
        printf("Decoded %d samples @ %d Hz\r\n",
               decExt.frameLength, decExt.samplingRate);
        *pcm_samples = decExt.frameLength * decExt.encodedChannels;
    } else {
        printf("Decode failed: %d\r\n", ret);
    }

    /* 步骤6: 释放资源 */
    free(pDecoderMem);

    return ret;
}
```

### 连续解码多帧

```c
int aac_decode_stream(const uint8_t *aac_stream, uint32_t stream_len,
                      int16_t *pcm_buf, uint32_t max_pcm_samples)
{
    tPVMP4AudioDecoderExternal decExt = {0};
    void *pDecoderMem;
    UInt32 memSize;
    Int ret;
    uint32_t total_samples = 0;
    uint32_t offset = 0;

    memSize = AACDecMemReq();
    pDecoderMem = malloc(memSize);
    if (!pDecoderMem) return -1;

    /* 初始化 (省略错误检查) */
    decExt.pOutputBuffer    = pcm_buf;
    decExt.desiredChannels  = 2;
    decExt.outputFormat     = OUTPUTFORMAT_16PCM_INTERLEAVED;
    decExt.aacPlusEnabled   = TRUE;
    decExt.aacPlusUpsamplingFactor = 2;

    AACInitDecoder(&decExt, pDecoderMem);

    /* 循环解码直到数据耗尽 */
    while (offset < stream_len) {
        decExt.pInputBuffer           = (UChar *)&aac_stream[offset];
        decExt.inputBufferCurrentLength = stream_len - offset;
        decExt.inputBufferUsedLength  = 0;

        ret = AACDecodeFrame(&decExt, pDecoderMem);
        if (ret != AAC_SUCCESS) {
            break;
        }

        offset += decExt.inputBufferUsedLength;
        total_samples += decExt.frameLength * decExt.encodedChannels;

        /* 检查输出缓冲区容量 */
        if (total_samples > max_pcm_samples) {
            break;
        }

        /* 更新输出缓冲区指针 */
        decExt.pOutputBuffer = &pcm_buf[total_samples];
    }

    free(pDecoderMem);
    return total_samples;
}
```

## 音频对象类型

解码器支持的 MPEG-4 音频对象类型 (定义于 `e_tMP4AudioObjectType.h`):

| 类型值 | 名称 | 说明 |
|-------|------|------|
| 1 | AAC_MAIN | AAC Main profile |
| 2 | AAC_LC | Low Complexity (最常用) |
| 3 | AAC_SSR | Scalable Sampling Rate |
| 4 | LTP | Long Term Prediction |
| 5 | SBR | Spectral Band Replication |
| 17 | ER_AAC_LC | Error Resilient AAC-LC |
| 23 | ER_AAC_LD | Error Resilient AAC-LD |
| 29 | PS | Parametric Stereo |

## 注意事项

1. **内存对齐**: 分配的解码器内存建议按 4 字节对齐

2. **输入缓冲区**: 解码器按帧处理数据，`inputBufferUsedLength` 指示实际消耗字节数

3. **AAC+ 输出**: 当启用 AAC+ 解码时，需要额外提供 `pOutputBuffer_plus` 缓冲区 (2048 样本)

4. **线程安全**: 解码器本身不保证线程安全，多线程使用时需自行同步

5. **帧长度**: 输出帧长度固定为 1024 样本/声道，与采样率无关

## 参考

- [PVMP4AudioDecoder API 头文件](../../workspase/BL618Claw/bouffalo_sdk/components/multimedia/aacdec/include/pvmp4audiodecoder_api.h)
- [PV Audio 类型定义](../../workspase/BL618Claw/bouffalo_sdk/components/multimedia/aacdec/include/pv_audio_type_defs.h)
- [MPEG-4 Audio Object Types](../../workspase/BL618Claw/bouffalo_sdk/components/multimedia/aacdec/include/e_tmp4audioobjecttype.h)
- ISO/IEC 14496-3:2001 - MPEG-4 音频编码标准
