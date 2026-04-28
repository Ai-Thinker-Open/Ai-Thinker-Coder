# BL618 音频编解码驱动

## 概述

BL618 音频驱动层位于 Bouffalo SDK 的 `multimedia` 组件中，主要包含两大模块：

- **SndBl616 声卡驱动**：`drv_snd_bl616` 组件，提供 I2S/PCMDM/PDM 音频接口的声卡抽象
- **AudioFlowctrlBridge 音频流桥接**：`audio_flowctrl_bridge` 组件，实现 SBC 解码到 PCM 的完整音频流处理

这两层在架构上服务于上层的 `smart_audio`，为语音播放、音频录制等场景提供稳定的底层支撑。

---

## SndBl616 声卡驱动

### 头文件

```c
#include <devices/drv_snd_bl616.h>
```

### 配置结构体

`snd_bl616_config_t` 用于初始化声卡驱动的增益参数：

```c
typedef struct {
    int audio_in_gain_list[3];   // 输入增益列表，支持 3 档增益配置
    int audio_out_gain_list[2];  // 输出增益列表，支持 2 档增益配置
} snd_bl616_config_t;
```

> `audio_in_gain_list` 和 `audio_out_gain_list` 分别对应模拟/数字输入增益和输出增益的校准值，实际取值需根据硬件设计调优。

### 注册与注销

```c
void snd_card_bl616_register(void *config);
void snd_card_bl616_unregister(void *config);
```

- `snd_card_bl616_register()`：根据传入的配置参数注册声卡驱动，完成 I2S/PCMDM/PDM 硬件资源初始化
- `snd_card_bl616_unregister()`：注销声卡，释放相关硬件资源

### I2S 输入/输出增益配置

I2S 接口的输入增益通过 `audio_in_gain_list` 配置，支持 3 级增益调节，适用于不同灵敏度的麦克风输入。输出增益通过 `audio_out_gain_list` 配置，支持 2 级输出功率调节，用于驱动不同阻抗的扬声器或功放。

声卡注册后，驱动会根据配置列表自动设置对应增益寄存器，无需用户手动干预。

---

## AudioFlowctrlBridge 音频流桥接

AudioFlowctrlBridge 实现了从 SBC 压缩音频数据到 PCM 原始音频数据的完整解码与播放链路，包含 `pcm_drv` 驱动抽象层和 `sbc2pcm` 高层接口两部分。

### PCM 驱动层

PCM 驱动抽象层 `pcm_drv` 定义了标准 PCM 设备的操作接口：

```c
struct pcm_drv {
    pcm_handle_t *(*pcm_open)(int mode, int samplerate, int channels, int format);
    int (*pcm_write)(pcm_handle_t pcm, const void *buf, unsigned int size);
    int (*pcm_start)(pcm_handle_t pcm);
    int (*pcm_stop)(pcm_handle_t pcm);
    int (*pcm_ioctl)(pcm_handle_t pcm, int cmd, void *arg);
    void (*pcm_close)(pcm_handle_t pcm);
};

const struct pcm_drv *pcm_drv_register(void);
```

- `pcm_open`：以指定模式、采样率、通道数和格式打开 PCM 设备，返回句柄
- `pcm_write`：向 PCM 设备写入音频数据（通常为 PCM 原始采样）
- `pcm_start` / `pcm_stop`：启动/停止 PCM 数据流
- `pcm_ioctl`：发送控制命令，可用于设置音量、切换数据源等
- `pcm_close`：关闭 PCM 设备，释放句柄
- `pcm_drv_register`：注册 PCM 驱动，返回驱动函数表

> `pcm_handle_t` 为 `void *` 类型句柄，用于唯一标识打开的 PCM 设备实例。

### PCM 数据格式

`format` 参数通常使用标准 ALSA 格式定义（如 `S16LE`、`S8` 等），`channels` 支持单声道（1）或立体声（2），`samplerate` 常见值为 8000/16000/44100/48000 Hz。

### sbc2pcm 库

`sbc2pcm` 是构建在 `pcm_drv` 之上的高层封装，专门用于处理 SBC 蓝牙音频解码场景。它内部集成了 SBC 解码器，并将解码后的 PCM 数据通过底层 PCM 驱动输出。

#### 句柄类型

```c
typedef struct sbc2pcm_handle *sbc2pcm_player_handle_t;
```

#### 打开与关闭

```c
sbc2pcm_player_handle_t sbc2pcm_player_open(int mode, int samplerate, int channels, int format);
void sbc2pcm_player_close(sbc2pcm_player_handle_t handle);
```

`mode` 参数指定播放模式，`samplerate` 和 `channels` 分别设置目标采样率和声道数，`format` 指定 PCM 采样格式。

#### 写入数据

```c
int sbc2pcm_player_write(sbc2pcm_player_handle_t handle, const void *buf, unsigned int size);
```

将 SBC 压缩数据写入解码队列，库内部会自动完成 SBC 解码并输出 PCM 数据。返回值通常为写入的字节数或错误码。

#### 启动与停止

```c
int sbc2pcm_player_start(sbc2pcm_player_handle_t handle);
int sbc2pcm_player_stop(sbc2pcm_player_handle_t handle);
```

`start` 启动内部解码任务，开始从队列消费数据并推送至 PCM 驱动；`stop` 停止解码任务，数据流冻结但句柄保持有效。

#### SBC 解码初始化

```c
int sbc2pcm_decode_init();
```

在启动 sbc2pcm 播放之前需要调用此函数初始化解码器上下文，通常在系统初始化阶段执行一次即可。

#### 事件机制

`sbc2pcm` 内部维护一个事件队列，通过 `PLAYER_EVENT_*` 系列事件类型通知上层：

| 事件类型 | 说明 |
|---|---|
| `PLAYER_EVENT_SBC_DATA` | 收到待解码的 SBC 数据 |
| `PLAYER_EVENT_OPEN` | PCM 设备已打开 |
| `PLAYER_EVENT_START` | 播放流已启动 |
| `PLAYER_EVENT_STOP` | 播放流已停止 |
| `PLAYER_EVENT_CLOSE` | PCM 设备已关闭 |

上层可以通过事件队列监听播放状态变化，实现同步控制逻辑。

#### pcm_ioctl 控制命令

通过 `pcm_ioctl` 可以向 PCM 驱动发送特定控制命令，常见的命令包括：

- 音量调节
- 静音切换
- 数据源切换
- 采样率变更

具体命令编号和参数格式由具体实现定义，通常 `cmd` 为命令枚举，`arg` 为参数结构体指针。

---

## 与 smart_audio 的关系

`smart_audio` 是 BL618 音频方案的上层应用框架，封装了音频播放、录制、编解码等业务逻辑。SndBl616 声卡驱动和 AudioFlowctrlBridge 共同构成其底层支撑：

```
smart_audio（上层应用）
    └── AudioFlowctrlBridge（sbc2pcm / pcm_drv）
            └── SndBl616 声卡驱动（drv_snd_bl616）
                    └── 硬件（I2S / PCMDM / PDM）
```

当 smart_audio 需要播放 SBC 音频流时，数据经 `sbc2pcm` 解码为 PCM，再由 `pcm_drv` 传递给 `snd_bl616` 声卡驱动，最终通过 I2S/PCMDM/PDM 接口输出到音频编解码器或功放。

声卡驱动的增益配置（`audio_in_gain_list` / `audio_out_gain_list`）也直接影响 smart_audio 的音频质量表现。

---

## 代码示例

### 注册声卡

```c
#include <devices/drv_snd_bl616.h>

void audio_init(void)
{
    snd_bl616_config_t cfg = {
        .audio_in_gain_list  = {0, 10, 20},  // 输入增益 3 档
        .audio_out_gain_list = {0, 15},      // 输出增益 2 档
    };

    snd_card_bl616_register(&cfg);
}
```

### 播放 PCM 数据（通过 sbc2pcm）

```c
#include "sbc2pcm.h"

void audio_playback_example(void)
{
    sbc2pcm_player_handle_t player;

    /* 初始化 SBC 解码器 */
    sbc2pcm_decode_init();

    /* 打开播放器：模式 0，16kHz 采样，单声道，S16LE 格式 */
    player = sbc2pcm_player_open(0, 16000, 1, 16);
    if (!player) {
        return;
    }

    /* 启动播放 */
    sbc2pcm_player_start(player);

    /* 写入 SBC 数据并自动解码播放 */
    uint8_t sbc_data[512];
    int len = read(sbc_fd, sbc_data, sizeof(sbc_data));
    if (len > 0) {
        sbc2pcm_player_write(player, sbc_data, len);
    }

    /* 停止并关闭 */
    sbc2pcm_player_stop(player);
    sbc2pcm_player_close(player);
}
```

### 直接使用 PCM 驱动播放 PCM

```c
#include "pcm_drv.h"

void pcm_playback_example(void)
{
    const struct pcm_drv *drv = pcm_drv_register();
    if (!drv) {
        return;
    }

    /* 打开 PCM：模式 0，48kHz 采样，立体声，S16LE */
    pcm_handle_t pcm = drv->pcm_open(0, 48000, 2, 16);
    if (!pcm) {
        return;
    }

    drv->pcm_start(pcm);

    /* 写入 PCM 原始数据 */
    int16_t pcm_samples[1024];
    int len = read(pcm_fd, pcm_samples, sizeof(pcm_samples));
    if (len > 0) {
        drv->pcm_write(pcm, pcm_samples, len);
    }

    /* 静音控制 */
    int mute = 0;
    drv->pcm_ioctl(pcm, 3, &mute);  /* 命令 3 设置静音状态 */

    drv->pcm_stop(pcm);
    drv->pcm_close(pcm);
}
```

### 注销声卡

```c
void audio_deinit(void)
{
    snd_bl616_config_t cfg = {0};
    snd_card_bl616_unregister(&cfg);
}
```

---

## 注意事项

1. **初始化顺序**：使用 sbc2pcm 前必须先调用 `sbc2pcm_decode_init()`，确保解码器上下文就绪。
2. **增益配置**：输入/输出增益列表的值应与硬件设计匹配，错误的增益可能导致声音失真或音量过大/过小。
3. **采样率匹配**：上层写入的 SBC 数据采样率应与 `sbc2pcm_player_open` 时指定的采样率一致，避免输出音调异常。
4. **线程安全**：`sbc2pcm` 内部使用独立任务和消息队列处理解码，不在中断上下文中调用 `sbc2pcm_player_write`。
5. **资源释放**：声卡注销前应确保所有 PCM 设备已关闭，避免使用已释放的硬件资源。

---

## 参考

- [drv_snd_bl616.h - SndBl616 声卡驱动头文件](../../../../workspase/BL618Claw/bouffalo_sdk/components/multimedia/drv_snd_bl616/include/devices/drv_snd_bl616.h)
- [pcm_drv.h - PCM 驱动抽象层头文件](../../../../workspase/BL618Claw/bouffalo_sdk/components/multimedia/audio_flowctrl_bridge/include/pcm_drv.h)
- [sbc2pcm.h - SBC 转 PCM 播放库头文件](../../../../workspase/BL618Claw/bouffalo_sdk/components/multimedia/audio_flowctrl_bridge/include/sbc2pcm.h)
