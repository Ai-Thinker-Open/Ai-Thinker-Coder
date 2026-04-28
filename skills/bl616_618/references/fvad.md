# FVAD 语音活动检测

## 概述

FVAD（FreeVAD）是一个基于 WebRTC 的语音活动检测（Voice Activity Detection，VAD）模块，专门用于检测音频流中的语音区间。该模块源自 WebRTC VAD 算法，后由 Daniel Pirch 提取为独立的开源库（libfvad），具有体积小、功耗低、检测准确等特点。

在 BL618 平台上，FVAD 通常作为语音识别系统的前端处理模块，负责从连续音频流中标记出含有语音的片段，供后续的语音识别、唤醒词检测等算法处理。通过准确的语音区间检测，可以有效减少无用数据的处理，降低系统功耗和计算负担。

## 关键类型

### Fvad 句柄

```c
typedef struct Fvad Fvad;
```

`Fvad` 是一个不透明句柄类型，表示一个 VAD 实例。开发者不需要了解其内部结构，只需通过 API 操作该句柄。所有 VAD 相关操作都需要传入由 `fvad_new()` 创建的实例指针。

## 核心 API

### fvad_new — 创建 VAD 实例

```c
Fvad *fvad_new(void);
```

创建并初始化一个新的 VAD 实例。该函数会分配必要的内存并设置默认参数（采样率 8000 Hz，模式 0）。

**返回值：** 成功返回指向新实例的指针，失败返回 `NULL`（通常是内存分配失败）。创建成功后需调用 `fvad_free()` 释放。

---

### fvad_free — 释放 VAD 实例

```c
void fvad_free(Fvad *inst);
```

释放指定 VAD 实例占用的动态内存。使用 `fvad_new()` 创建的实例最终都应调用此函数释放，以防内存泄漏。

**参数说明：**
- `inst` — `fvad_new()` 返回的实例指针

---

### fvad_reset — 重置 VAD 状态

```c
void fvad_reset(Fvad *inst);
```

重新初始化 VAD 实例，清除所有内部状态，并将模式和采样率恢复为默认值（模式 0，采样率 8000 Hz）。与 `fvad_free()` + `fvad_new()` 相比，`fvad_reset()` 避免了重新分配内存的开销，适用于需要复用同一实例的场景。

**参数说明：**
- `inst` — VAD 实例指针

---

### fvad_process — 检测语音活动

```c
int fvad_process(Fvad *inst, const int16_t *frame, size_t length);
```

对一帧音频数据进行语音活动检测，是 VAD 的核心功能。

**参数说明：**
- `inst` — VAD 实例指针
- `frame` — 指向 PCM 16-bit 有符号样本数组的指针
- `length` — 样本数量，必须对应 10ms、20ms 或 30ms 的帧长

**采样率与帧长对应关系：**

| 采样率 (Hz) | 10ms 帧长 | 20ms 帧长 | 30ms 帧长 |
|-------------|-----------|-----------|-----------|
| 8000        | 80        | 160       | 240       |
| 16000       | 160       | 320       | 480       |
| 32000       | 320       | 640       | 960       |
| 48000       | 480       | 960       | 1440      |

**返回值：**
- `1` — 检测到语音（Active Voice）
- `0` — 未检测到语音（Non-active Voice）
- `-1` — 无效的帧长度（length 不符合要求）

---

### fvad_set_mode — 设置检测模式

```c
int fvad_set_mode(Fvad *inst, int mode);
```

设置 VAD 的激进程度（Aggressiveness Mode）。模式越高，对语音的判定越严格，即返回 1 的条件越苛刻，漏报概率增加，但误报概率降低。

**参数说明：**
- `inst` — VAD 实例指针
- `mode` — 激进模式，取值 0~3

**模式说明：**

| 模式 | 名称           | 说明                                           |
|------|----------------|------------------------------------------------|
| 0    | Quality        | 最高质量，最低漏报率，默认模式                  |
| 1    | Low Bitrate    | 低码率场景，减少误报                            |
| 2    | Aggressive     | 激进模式，更严格筛选                            |
| 3    | Very Aggressive | 最高激进，语音确认门槛最高                       |

**返回值：** 成功返回 0，失败返回 -1（mode 无效）。

**场景选择建议：**
- 通用场景（语音识别前端）：推荐模式 0 或 1
- 噪声环境：推荐模式 2 或 3
- 安静环境且要求高召回率：推荐模式 0

---

### fvad_set_sample_rate — 设置采样率

```c
int fvad_set_sample_rate(Fvad *inst, int sample_rate);
```

设置输入音频的采样率。内部处理统一在 8000 Hz 进行，高于 8000 Hz 的输入数据会被自动下采样。

**参数说明：**
- `inst` — VAD 实例指针
- `sample_rate` — 采样率，单位 Hz，有效值：8000、16000、32000、48000

**返回值：** 成功返回 0，失败返回 -1（采样率无效）。

**注意：** 默认采样率为 8000 Hz。建议在实际使用前根据输入音频格式显式设置正确的采样率，以获得准确的检测结果。

## 输入要求

FVAD 对输入音频有明确格式要求：

- **数据类型**：PCM 16-bit 有符号整数（`int16_t`）
- **帧长度**：仅支持 10ms、20ms、30ms 三种帧长
- **采样率**：8000、16000、32000、48000 Hz
- **声道**：单声道（Mono）

输入数据必须符合上述规范，否则 `fvad_process()` 将返回 -1。

## 使用流程

典型的 FVAD 使用流程如下：

```
1. fvad_new()           创建 VAD 实例
2. fvad_set_sample_rate() 设置采样率
3. fvad_set_mode()      (可选) 设置检测模式
4. fvad_process()       循环处理每帧音频
5. fvad_free()          释放实例
```

## 代码示例

以下示例展示如何在 BL618 上使用 FVAD 进行语音活动检测：

```c
#include "fvad.h"
#include <stdio.h>
#include <stddef.h>

/* 假设音频参数：16kHz 采样率，20ms 帧长 */
#define SAMPLE_RATE     16000
#define FRAME_DURATION  20  /* ms */
#define FRAME_SIZE      (SAMPLE_RATE * FRAME_DURATION / 1000)  /* 320 samples */

int vad_example(const int16_t *audio_buffer, size_t num_frames)
{
    /* 1. 创建 VAD 实例 */
    Fvad *vad = fvad_new();
    if (vad == NULL) {
        printf("Failed to create VAD instance\r\n");
        return -1;
    }

    /* 2. 设置采样率为 16kHz */
    if (fvad_set_sample_rate(vad, SAMPLE_RATE) != 0) {
        printf("Invalid sample rate: %d\r\n", SAMPLE_RATE);
        fvad_free(vad);
        return -1;
    }

    /* 3. 设置检测模式（0: 质量优先） */
    if (fvad_set_mode(vad, 0) != 0) {
        printf("Invalid mode\r\n");
        fvad_free(vad);
        return -1;
    }

    /* 4. 循环处理每帧音频 */
    int speech_frames = 0;
    for (size_t i = 0; i < num_frames; i++) {
        const int16_t *frame = audio_buffer + i * FRAME_SIZE;
        int result = fvad_process(vad, frame, FRAME_SIZE);

        if (result < 0) {
            printf("Frame %zu: invalid length\r\n", i);
            continue;
        }

        if (result == 1) {
            printf("Frame %zu: speech detected\r\n", i);
            speech_frames++;
        } else {
            printf("Frame %zu: silence\r\n", i);
        }
    }

    /* 5. 释放 VAD 实例 */
    fvad_free(vad);

    printf("Total speech frames: %d / %zu\r\n", speech_frames, num_frames);
    return 0;
}
```

### 实时音频流处理示例

在实际应用中，音频数据通常来自麦克风或音频输入接口（如 I2S）。下面是一个简化的实时处理框架：

```c
#include "fvad.h"

/* 音频缓冲区大小计算：16kHz * 20ms = 320 samples */
#define FRAME_SIZE  320

void audio_vad_task(void *param)
{
    Fvad *vad = fvad_new();
    fvad_set_sample_rate(vad, 16000);
    fvad_set_mode(vad, 1);  /* 低码率模式，减少误报 */

    int16_t pcm_frame[FRAME_SIZE];

    while (1) {
        /* 从音频接口读取一帧数据（伪代码） */
        // audio_read_frame(pcm_frame, FRAME_SIZE);

        int is_speech = fvad_process(vad, pcm_frame, FRAME_SIZE);

        if (is_speech == 1) {
            /* 检测到语音，可触发后续处理如：
             * - 唤醒词检测
             * - 语音识别
             * - 录音存储
             */
        }
    }

    fvad_free(vad);
}
```

## 应用场景

### 语音识别前端

在离线语音识别或关键词唤醒系统中，FVAD 用于检测语音段，将连续的音频流切分为独立的语音片段。只有检测到语音时才启动识别算法，可以显著降低功耗和误触发率，是嵌入式语音交互的标准前端处理。

### 通话静音检测

在 VoIP 电话或视频会议应用中，FVAD 可用于检测用户是否在说话。当检测到静音时，系统可以选择不传输或压缩音频数据，节省网络带宽。在通话质量监控中，静音检测也有助于生成通话质量报告。

### 录音触发

在需要录音笔、语音备忘、环境监听等场景中，VAD 用于检测声音活动以触发录音开始和结束。相比于持续录音，触发式录音可以大幅节省存储空间和电量，同时减少后期需要处理的无用音频数据。

## 性能特性

- **低计算开销**：基于短时能量和频域特征的简单判决算法，适合嵌入式 MCU
- **低内存占用**：单实例内存占用极小，适合资源受限的系统
- **低延迟**：逐帧处理，检测延迟约为单帧时长（10~30ms）
- **独立性强**：纯 C 实现，无外部依赖，易于移植

## 注意事项

1. **帧对齐**：输入帧长度必须精确匹配规范，不支持变长帧。
2. **采样率匹配**：必须使用实际音频采样率进行设置，否则检测结果不准确。
3. **模式选择**：应根据实际噪声环境选择合适的检测模式，噪声越大宜选用较高模式。
4. **线程安全**：`fvad_process()` 不是线程安全的，多线程使用时需自行同步。
5. **重置时机**：更换音频流或检测场景时，建议调用 `fvad_reset()` 清除内部状态。

## 参考

- [libfvad 官方仓库](https://github.com/dpirch/libfvad)
- [WebRTC VAD 算法文档](https://webrtc.github.io/webrtc-org/audio/)
- `/home/seahi/workspase/BL618Claw/bouffalo_sdk/components/multimedia/libfvad/include/fvad.h`
