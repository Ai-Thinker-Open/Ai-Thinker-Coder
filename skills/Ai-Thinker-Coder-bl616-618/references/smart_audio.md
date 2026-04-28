# Smart Audio 音频框架

## 概述

Smart Audio 是 BL618 芯片的统一音频播放框架，整合了本地音乐、提示音、蓝牙 A2DP、蓝牙 HFP 四种音频源。该框架提供音量管理和播放状态机，支持通过回调机制通知应用层播放状态变化。

## 播放源类型

| 类型 | 枚举值 | 说明 |
|------|--------|------|
| SMTAUDIO_ONLINE_MUSIC | MEDIA_MUSIC | 在线音乐播放 |
| SMTAUDIO_LOCAL_PLAY | MEDIA_SYSTEM | 本地提示音播放 |
| SMTAUDIO_BT_A2DP | 102 | 蓝牙 A2DP 音乐 |
| SMTAUDIO_BT_HFP | 103 | 蓝牙 HFP 通话 |

```c
typedef enum {
    SMTAUDIO_ONLINE_MUSIC = MEDIA_MUSIC,
    SMTAUDIO_LOCAL_PLAY   = MEDIA_SYSTEM,
    SMTAUDIO_BT_A2DP     = 102,
    SMTAUDIO_BT_HFP      = 103,
    SMTAUDIO_PLAY_TYPE_NUM = 4,
    SMTAUDIO_TYPE_ALL     = 255,
} smtaudio_player_type_t;
```

## 播放状态

```c
typedef enum {
    SMTAUDIO_STATE_PLAYING,  // 正在播放
    SMTAUDIO_STATE_PAUSE,    // 暂停
    SMTAUDIO_STATE_STOP,     // 停止
    SMTAUDIO_STATE_MUTE,     // 静音
    SMTAUDIO_STATE_NOINIT,   // 未初始化
} smtaudio_state_t;
```

## 子状态机

每种播放源有独立的子状态：

```c
typedef enum {
    // 在线音乐子状态
    SMTAUDIO_SUBSTATE_ONLINE_PLAYING,
    SMTAUDIO_SUBSTATE_ONLINE_PAUSE,
    SMTAUDIO_SUBSTATE_ONLINE_STOP,

    // 本地提示音子状态
    SMTAUDIO_SUBSTATE_LOCAL_PLAYING,
    SMTAUDIO_SUBSTATE_LOCAL_PAUSE,
    SMTAUDIO_SUBSTATE_LOCAL_STOP,

    // 蓝牙 A2DP 子状态
    SMTAUDIO_SUBSTATE_BT_A2DP_PLAYING,
    SMTAUDIO_SUBSTATE_BT_A2DP_PAUSE,
    SMTAUDIO_SUBSTATE_BT_A2DP_STOP,

    // 蓝牙 HFP 子状态
    SMTAUDIO_SUBSTATE_BT_HFP_PLAYING,
    SMTAUDIO_SUBSTATE_BT_HFP_PAUSE,
    SMTAUDIO_SUBSTATE_BT_HFP_STOP,

    SMTAUDIO_SUBSTATE_MUTE,
} smtaudio_sub_state_t;
```

## 核心常量

| 常量 | 值 | 说明 |
|------|-----|------|
| VOLUME_SAVE_KV_NAME | "volume" | 音量持久化键名 |
| SMART_AUDIO_DEFAULT_VOLUME | 60 | 默认音量值 |
| INTERRUPT_REASON_BY_USER | 255 | 用户中断原因码 |

## 核心 API

### 初始化

```c
int8_t smtaudio_init(audio_evt_t audio_evt_cb);
```

### 注册播放源

```c
int8_t smtaudio_register_local_play(uint8_t min_vol, uint8_t *aef_conf, 
                                     size_t aef_conf_size, float speed, int resample);
int8_t smtaudio_register_bt_a2dp(uint8_t min_vol, uint8_t *aef_conf,
                                  size_t aef_conf_size, float speed, int resample);
int8_t smtaudio_register_bt_hfp(uint8_t min_vol, uint8_t *aef_conf,
                                 size_t aef_conf_size, float speed, int resample);
int8_t smtaudio_register_online_music(uint8_t min_vol, uint8_t *aef_conf,
                                       size_t aef_conf_size, float speed, int resample);
```

### 播放控制

```c
int8_t smtaudio_start(int type, char *url, uint64_t seek_time, uint8_t resume);
int8_t smtaudio_pause(void);
int8_t smtaudio_resume(void);
int8_t smtaudio_stop(int type);
int8_t smtaudio_mute(void);
```

### 音量控制

```c
int8_t smtaudio_vol_set(int16_t vol);    // 设置音量 (0-100)
int8_t smtaudio_vol_get(void);           // 获取当前音量
int8_t smtaudio_vol_up(int16_t vol);     // 音量增加
int8_t smtaudio_vol_down(int16_t vol);   // 音量减少
int8_t smtaudio_vol_config(audio_vol_config_t *vol_config); // 配置音量映射
```

### 状态查询

```c
smtaudio_state_t smtaudio_get_state(void);
smtaudio_player_type_t smtaudio_get_play_type(void);
int8_t smtaudio_info(int type, smtaudio_play_time_t *t);
```

### 低功耗管理

```c
int8_t smtaudio_lpm(uint8_t state);      // 设置低功耗模式
int smtaudio_enter_lpm_check(void);      // 检查是否可以进入低功耗
```

## 事件回调

```c
typedef void (*audio_evt_t)(int type, smtaudio_player_evtid_t evt_id);

typedef enum {
    SMTAUDIO_PLAYER_EVENT_ERROR,
    SMTAUDIO_PLAYER_EVENT_START,
    SMTAUDIO_PLAYER_EVENT_STOP,
    SMTAUDIO_PLAYER_EVENT_RESUME,
    SMTAUDIO_PLAYER_EVENT_UNDER_RUN,
    SMTAUDIO_PLAYER_EVENT_OVER_RUN,
    SMTAUDIO_PLAYER_EVENT_PAUSE,
} smtaudio_player_evtid_t;
```

## 播放时间信息

```c
typedef struct {
    uint64_t duration;  // 总时长（毫秒）
    uint64_t curtime;   // 当前播放位置（毫秒）
} smtaudio_play_time_t;
```

## 音量配置结构

```c
typedef struct _audio_vol_config {
    int32_t db_min;    // 最小分贝值
    int32_t db_max;    // 最大分贝值
    uint8_t *map;      // 音量映射表（大小 101）
} audio_vol_config_t;
```

## 与蓝牙模块的关系

### 蓝牙 A2DP 接口

```c
int msp_app_bt_a2dp_connect(uint8_t remote_addr[BT_BD_ADDR_LEN]);
int msp_app_bt_a2dp_disconnect(void);
int msp_app_bt_a2dp_get_connect_status(void);
int msp_app_bt_avrcp_send_passthrouth_cmd(msp_app_avrcp_cmd_type_t cmd_type);
int msp_app_bt_avrcp_change_vol(uint8_t vol);
int msp_app_bt_a2dp_register_cb(msp_app_bt_callback_t callback);
```

AVRCP 命令类型：

```c
typedef enum {
    MSP_APP_BT_AVRCP_CMD_PLAY = 0,
    MSP_APP_BT_AVRCP_CMD_PAUSE,
    MSP_APP_BT_AVRCP_CMD_FORWARD,
    MSP_APP_BT_AVRCP_CMD_BACKWARD,
    MSP_APP_BT_AVRCP_CMD_FAST_FORWARD,
    MSP_APP_BT_AVRCP_CMD_REWIND,
    MSP_APP_BT_AVRCP_CMD_STOP,
    MSP_APP_BT_AVRCP_CMD_VOL_UP,
    MSP_APP_BT_AVRCP_CMD_VOL_DOWN,
} msp_app_avrcp_cmd_type_t;
```

### 蓝牙 HFP 接口

```c
int32_t msp_app_bt_hfp_reg_callback(MSP_APP_BT_HFP_IMPL_CB_FUNC_T *callback);
int32_t msp_app_bt_hfp_connect(const char *mac);
int32_t msp_app_bt_hfp_disconnect(const char *mac);
int32_t msp_app_bt_hfp_connect_audio(const char *mac);
int32_t msp_app_bt_hfp_disconnect_audio(const char *mac);
int32_t msp_app_bt_hfp_send_command(MSP_APP_BT_HFP_COMMAND_T command);
int32_t msp_app_bt_hfp_dial(char *number);
int32_t msp_app_bt_hfp_get_call_status(void);
int32_t msp_app_bt_hfp_volume_update(int type, int volume);
```

HFP 状态枚举：

```c
typedef enum hfp_call_status {
    CALL_NONE = 0,
    CALL_NO_PROGRESS,
    CALL_IN_PROGRESS,
    CALL_INCOMING,
    CALL_OUTGOING_DIALING,
} msp_app_bt_hfp_call_status_t;

typedef enum {
    MSP_APP_BT_HFP_CONNECTION_STATE_DISCONNECTED = 0,
    MSP_APP_BT_HFP_CONNECTION_STATE_CONNECTING,
    MSP_APP_BT_HFP_CONNECTION_STATE_CONNECTED,
    MSP_APP_BT_HFP_CONNECTION_STATE_SLC_CONNECTED,
    MSP_APP_BT_HFP_CONNECTION_STATE_DISCONNECTING,
} MSP_APP_BT_HFP_STATE_T;

typedef enum {
    MSP_APP_BT_HFP_AUDIO_STATE_DISCONNECTED = 0,
    MSP_APP_BT_HFP_AUDIO_STATE_CONNECTING,
    MSP_APP_BT_HFP_AUDIO_STATE_CONNECTED,
    MSP_APP_BT_HFP_AUDIO_STATE_CONNECTED_MSBC,
} MSP_APP_BT_HFP_AUDIO_STATE_T;
```

## 代码示例

### 示例一：初始化

```c
#include "smart_audio.h"

void audio_event_callback(int type, smtaudio_player_evtid_t evt_id)
{
    switch (evt_id) {
        case SMTAUDIO_PLAYER_EVENT_START:
            printf("[Audio] Play started\r\n");
            break;
        case SMTAUDIO_PLAYER_EVENT_STOP:
            printf("[Audio] Play stopped\r\n");
            break;
    }
}

void smart_audio_init(void)
{
    // 初始化
    smtaudio_init(audio_event_callback);

    // 注册播放源
    smtaudio_register_local_play(5, NULL, 0, 1.0f, 48000);
    smtaudio_register_bt_a2dp(5, NULL, 0, 1.0f, 48000);
    smtaudio_register_bt_hfp(5, NULL, 0, 1.0f, 16000);
    smtaudio_register_online_music(5, NULL, 0, 1.0f, 48000);

    // 设置默认音量
    smtaudio_vol_set(SMART_AUDIO_DEFAULT_VOLUME);
}
```

### 示例二：播放本地提示音

```c
void play_notification(void)
{
    // 播放本地提示音
    smtaudio_start(SMTAUDIO_LOCAL_PLAY, "/system/notification.wav", 0, 0);

    // 获取播放信息
    smtaudio_play_time_t time_info;
    smtaudio_info(SMTAUDIO_LOCAL_PLAY, &time_info);
    printf("Duration: %llu ms\r\n", time_info.duration);

    // 停止播放
    smtaudio_stop(SMTAUDIO_LOCAL_PLAY);
}
```

### 示例三：蓝牙 A2DP 播放

```c
#include "bt/msp_app_bt.h"

void bt_a2dp_play(uint8_t *remote_addr)
{
    // 连接蓝牙设备
    msp_app_bt_a2dp_connect(remote_addr);

    // 播放蓝牙音乐
    smtaudio_start(SMTAUDIO_BT_A2DP, NULL, 0, 1);

    // 设置音量
    smtaudio_vol_set(70);

    // 发送 AVRCP 命令
    msp_app_bt_avrcp_send_passthrouth_cmd(MSP_APP_BT_AVRCP_CMD_PAUSE);

    // 恢复播放
    smtaudio_resume();

    // 获取当前状态
    smtaudio_state_t state = smtaudio_get_state();
    smtaudio_player_type_t type = smtaudio_get_play_type();

    // 断开连接
    msp_app_bt_a2dp_disconnect();
}
```

### 示例四：蓝牙 HFP 通话

```c
void hfp_call_callbacks(MSP_APP_BT_HFP_STATE_T state, const char *mac)
{
    printf("[HFP] State: %d, MAC: %s\r\n", state, mac);
}

void bt_hfp_call(const char *mac)
{
    MSP_APP_BT_HFP_IMPL_CB_FUNC_T callbacks = {
        .hfpStateChangedCB = hfp_call_callbacks,
        .hfpAudioStateCB = NULL,
        .hfpVolumeChangedCB = NULL,
        .hfpRingIndCB = NULL,
    };

    // 注册回调并连接
    msp_app_bt_hfp_reg_callback(&callbacks);
    msp_app_bt_hfp_connect(mac);
    msp_app_bt_hfp_connect_audio(mac);

    // 设置音量
    msp_app_bt_hfp_volume_update(0, 10);

    // 拨打电话
    msp_app_bt_hfp_dial("10086");

    // 通话结束挂断
    msp_app_bt_hfp_send_command(MSP_APP_BT_HFP_COMMAND_TYPE_TERMINATE);
    msp_app_bt_hfp_disconnect(mac);
}
```

### 示例五：音量控制

```c
void volume_control(void)
{
    // 设置音量
    smtaudio_vol_set(80);

    // 获取当前音量
    int8_t vol = smtaudio_vol_get();
    printf("Current volume: %d\r\n", vol);

    // 音量调节
    smtaudio_vol_up(10);
    smtaudio_vol_down(5);

    // 静音
    smtaudio_mute();
    smtaudio_resume();

    // 获取播放状态
    smtaudio_state_t state = smtaudio_get_state();
}
```

### 示例六：自定义音量映射

```c
void volume_config_example(void)
{
    static uint8_t volume_db_map[101];

    // 构建自定义映射表
    for (int i = 0; i <= 100; i++) {
        volume_db_map[i] = i;  // 线性映射
    }

    audio_vol_config_t vol_config = {
        .db_min = -60,
        .db_max = 0,
        .map = volume_db_map,
    };

    smtaudio_vol_config(&vol_config);
}
```

## 参考

- [smart_audio.h 头文件源码](../workspase/BL618Claw/bouffalo_sdk/components/multimedia/smart_audio_bl616/include/smart_audio.h)
- [msp_app_bt.h 蓝牙接口头文件](../workspase/BL618Claw/bouffalo_sdk/components/multimedia/smart_audio_bl616/include/bt/msp_app_bt.h)
- [Bouffalo SDK 多媒体组件](../workspase/BL618Claw/bouffalo_sdk/components/multimedia)
- [BL618 芯片技术手册](https://www.bouffalolab.com)
