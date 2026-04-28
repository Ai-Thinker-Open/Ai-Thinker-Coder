# Bluetooth AVRCP 技术文档

## 1. 概述

AVRCP（Audio/Video Remote Control Profile，音视频远程控制配置文件）是蓝牙协议栈中用于控制 A2DP（Advanced Audio Distribution Profile）音频流播放的配置文件。通过 AVRCP，用户可以在蓝牙设备上远程控制另一设备的音乐播放功能，例如播放、暂停、停止、上一曲、下一曲、音量调节等操作。

AVRCP 基于 AVCTP（Audio/Video Control Transport Protocol，音视频控制传输协议）进行数据传输。AVCTP 定义了设备之间控制命令和响应消息的传输格式，而 AVRCP 则定义了这些命令的语义和交互流程。

在蓝牙协议栈中，AVRCP 通常与 A2DP 配合使用：
- **A2DP**：负责音频数据的传输，决定音质和编码格式
- **AVRCP**：负责音频播放的控制，实现用户交互操作

两者协同工作，AVRCP 发送播放控制命令，A2DP 负责传输实际的音频流数据，从而实现完整的蓝牙音乐播放功能。

## 2. AVCTP 传输层

AVCTP 是 AVRCP 的底层传输协议，定义了命令和响应的封装格式。

### 2.1 AVCTP 关键参数

| 参数 | 值 | 说明 |
|------|-----|------|
| L2CAP PSM | 0x0017 | AVCTP 使用的 L2CAP 协议服务通道 |
| PID | 0x0e11 | 协议标识符 |
| CR Command | 0 | 命令报文 |
| CR Response | 1 | 响应报文 |

### 2.2 AVCTP 数据包类型

AVCTP 支持四种数据包类型，用于处理不同长度的数据：

| 类型 | 值 | 说明 |
|------|-----|------|
| SINGLE | 0x0 | 单包，数据完整在一个数据包中 |
| START | 0x1 | 分包起始包 |
| CONTINUE | 0x2 | 分包中间包 |
| END | 0x3 | 分包结束包 |

当 AVRCP 命令或响应数据超过 L2CAP MTU 大小时，AVCTP 会自动将数据拆分多个数据包进行传输，接收端则将多个数据包重新组装成完整数据。

## 3. AVRCP 命令类型（ctype）

AVRCP 定义了多种命令类型，用于区分不同的操作意图：

| 命令类型 | 值 | 说明 |
|----------|-----|------|
| CONTROL | 0x00 | 控制命令，用于执行播放控制操作 |
| STATUS | 0x01 | 状态查询，用于获取当前播放状态 |
| SPECIFIC_INQUIRY | 0x02 | 特定查询 |
| NOTIFY | 0x03 | 事件通知注册，请求对方主动推送特定事件 |
| GENERAL_INQUIRY | 0x04 | 通用查询 |

### 3.1 CONTROL 命令

CONTROL 命令用于执行具体的播放控制操作，如播放、暂停、停止等。这类命令通常需要目标设备执行相应的动作并返回响应。

### 3.2 STATUS 命令

STATUS 命令用于查询目标设备的当前状态，如播放状态、当前曲目信息、播放位置等。目标设备收到此类命令后应返回包含当前状态信息的响应。

### 3.3 NOTIFY 命令

NOTIFY 命令用于注册事件通知。当设备发送 NOTIFY 命令订阅某个事件后，只要该事件发生，目标设备就会主动向订阅者推送通知。这是一种事件驱动模式，避免了客户端频繁轮询。

## 4. AVRCP 响应类型

AVRCP 命令的响应有以下几种类型：

| 响应类型 | 值 | 说明 |
|----------|-----|------|
| NOT_IMPLEMENTED | 0x08 | 命令不被支持或无法处理 |
| ACCEPTED | 0x09 | 命令已被接受并成功执行 |
| REJECTED | 0x0A | 命令被拒绝，通常由于参数错误或状态不对 |
| IN_TRANSITION | 0x0B | 状态正在转换中 |
| IMPLEMENTED | 0x0C | 命令已实现（用于查询响应） |
| CHANGED | 0x0D | 状态已改变 |
| INTERIM | 0x0F | 中间响应，表示命令正在处理中 |

INTERIM 响应通常用于 NOTIFY 命令，当目标设备收到订阅请求后，首先返回一个 INTERIM 响应表示请求已收到，实际的事件通知会在后续发送。

## 5. AVRCP 操作码（Opcode）

Opcode 定义了 AVRCP 命令的具体操作类型：

| 操作码 | 值 | 说明 |
|--------|-----|------|
| UNIT_INFO | 0x30 | 单元信息查询 |
| SUBUNIT_INFO | 0x31 | 子单元信息查询 |
| PASS_THROUGH | 0x7C | 穿透命令，用于传输按键/控制操作 |
| VENDOR_DEPENDENT | 0x00 | 厂商自定义命令 |

### 5.1 PASS_THROUGH 命令

PASS_THROUGH 是最常用的 Opcode，用于传输标准的播放控制命令。所有播放、暂停、停止、上下曲等操作都通过 PASS_THROUGH 命令发送。

### 5.2 VENDOR_DEPENDENT 命令

VENDOR_DEPENDENT 用于传输厂商自定义的命令扩展，可用于实现非标准的控制功能或传输特定厂商的元数据。

## 6. 按键状态（Pass-through State）

在 PASS_THROUGH 命令中，每个按键操作包含两个状态：

| 状态 | 值 | 说明 |
|------|-----|------|
| PRESSED | 0x00 | 按键按下 |
| RELEASED | 0x01 | 按键释放 |

典型的用户交互流程是：按下按键时发送 PRESSED 状态，释放按键时发送 RELEASED 状态。这种设计允许目标设备区分短按和长按操作。

## 7. 播放状态

AVRCP 定义了以下播放状态：

| 播放状态 | 值 | 说明 |
|----------|-----|------|
| STOPPED | 0x00 | 停止播放 |
| PLAYING | 0x01 | 正在播放 |
| PAUSED | 0x02 | 暂停播放 |
| FWD_SEEK | 0x03 | 正在快进 |
| REV_SEEK | 0x04 | 正在快退 |
| ERROR | 0xFF | 播放错误 |

## 8. 按键操作 ID

PASS_THROUGH 命令中的操作 ID 定义了具体的控制功能：

| 操作 ID | 值 | 说明 |
|---------|-----|------|
| KEY_VOL_UP | 0x41 | 音量增加 |
| KEY_VOL_DOWN | 0x42 | 音量减少 |
| KEY_PLAY | 0x44 | 播放 |
| KEY_STOP | 0x45 | 停止 |
| KEY_PAUSE | 0x46 | 暂停 |
| KEY_REWIND | 0x48 | 快退/上一曲 |
| KEY_FAST_FORWARD | 0x49 | 快进/下一曲 |
| KEY_FORWARD | 0x4B | 下一曲 |
| KEY_BACKWARD | 0x4C | 上一曲 |

## 9. 事件通知

AVRCP 支持多种事件通知类型，设备可以通过 NOTIFY 命令订阅这些事件：

| 事件 ID | 值 | 说明 |
|----------|-----|------|
| PLAYBACK_STATUS_CHANGED | 0x01 | 播放状态变化 |
| TRACK_CHANGED | 0x02 | 曲目变化 |
| TRACK_REACHED_END | 0x03 | 曲目播放到达结尾 |
| TRACK_REACHED_START | 0x04 | 曲目播放到达开头 |
| PLAYBACK_POS_CHANGED | 0x05 | 播放位置变化 |
| BATT_STATUS_CHANGED | 0x06 | 电池状态变化 |
| SYSTEM_STATUS_CHANGED | 0x07 | 系统状态变化 |
| PLAYER_APPLICATION_SETTING_CHANGED | 0x08 | 播放器设置变化 |
| NOW_PLAYING_CONTENT_CHANGED | 0x09 | 当前播放内容变化 |
| AVAILABLE_PLAYERS_CHANGED | 0x0A | 可用播放器列表变化 |
| ADDRESSED_PLAYER_CHANGED | 0x0B | 当前寻址的播放器变化 |
| UIDS_CHANGED | 0x0C | UIDs 列表变化 |
| VOLUME_CHANGED | 0x0D | 音量变化 |

### 9.1 常用事件说明

**PLAYBACK_STATUS_CHANGED (0x01)**
当播放状态发生变化时触发，如从播放变为暂停、从暂停变为播放等。事件数据包含新的播放状态值。

**TRACK_CHANGED (0x02)**
当当前播放的曲目发生变化时触发，如切换到下一曲或上一曲。事件数据包含新曲目的标识信息。

**PLAYBACK_POS_CHANGED (0x05)**
当播放位置发生变化时触发。该事件用于同步播放进度，事件数据包含当前的播放位置（以毫秒为单位）。

**VOLUME_CHANGED (0x0D)**
当音量发生变化时触发。事件数据包含当前的音量值（0x00-0x7F）。注意音量值仅使用低 7 位。

## 10. Vendor 命令 PDU ID

在 VENDOR_DEPENDENT 命令中，使用 PDU ID 来标识不同的厂商自定义操作：

| PDU ID | 值 | 命令类型 | 说明 |
|--------|-----|----------|------|
| GET_CAPABILITIES | 0x10 | STATUS | 获取设备能力 |
| GET_ELEMENT_ATTRS | 0x20 | STATUS | 获取媒体元素属性 |
| GET_PLAY_STATUS | 0x30 | STATUS | 获取播放状态 |
| REGISTER_NOTIFICATION | 0x31 | NOTIFY | 注册事件通知 |
| REQUEST_CONTINUE_RSP | 0x40 | CONTROL | 请求继续响应 |
| ABORT_CONTINUE_RSP | 0x41 | CONTROL | 中止继续响应 |
| SET_ABSOLUTE_VOLUME | 0x50 | CONTROL | 设置绝对音量 |

### 10.1 绝对音量控制

AVRCP 支持绝对音量控制，通过 SET_ABSOLUTE_VOLUME 命令可以直接设置目标设备的音量值。音量值范围为 0x00-0x7F（低 7 位有效）。

## 11. 能力 ID

使用 GET_CAPABILITIES 命令可以查询目标设备支持的能力：

| 能力 ID | 值 | 说明 |
|---------|-----|------|
| COMPANY_ID | 0x02 | 公司标识列表 |
| EVENTS_SUPPORTED | 0x03 | 支持的事件列表 |

## 12. 与 A2DP 的协同关系

AVRCP 和 A2DP 是蓝牙音频应用中两个核心配置文件，它们各自承担不同的职责：

### 12.1 功能分工

- **A2DP（Advanced Audio Distribution Profile）**
  - 负责音频数据的传输
  - 定义音频编码格式（如 SBC、AAC、aptX 等）
  - 管理音频流的建立和控制
  - 确保音频数据的高质量传输

- **AVRCP（Audio/Video Remote Control Profile）**
  - 负责播放控制命令的传输
  - 实现播放、暂停、停止、切换曲目等控制功能
  - 提供播放状态和曲目信息的查询
  - 支持音量控制和事件通知

### 12.2 协同工作流程

当用户按下播放键时，AVRCP 负责将播放命令传输到目标设备。目标设备收到命令后：
1. 解析播放命令
2. 启动/恢复 A2DP 音频流传输
3. 更新播放状态
4. 通过 AVRCP 发送状态变化通知

两个协议通过共享的蓝牙连接进行通信，但使用不同的 L2CAP 通道：
- A2DP 使用音频传输通道
- AVRCP 使用控制通道（PSM: 0x0017）

## 13. 代码示例

### 13.1 AVRCP 播放控制请求

以下示例展示如何使用 PASS_THROUGH 命令发送播放控制请求：

```c
#include "bluetooth/avrcp.h"

/* 发送播放命令 */
int send_play_command(struct bt_avctp *session)
{
    uint8_t released = PASTHR_STATE_PRESSED;  /* 按键按下 */
    uint8_t opid = AVRCP_KEY_PLAY;            /* 播放操作 */

    /* 发送 PASS_THROUGH 命令 */
    int ret = avrcp_pasthr_cmd(session, released, opid);
    if (ret < 0) {
        /* 发送失败处理 */
        return ret;
    }

    /* 发送按键释放状态 */
    released = PASTHR_STATE_RELEASED;
    ret = avrcp_pasthr_cmd(session, released, opid);

    return ret;
}

/* 发送暂停命令 */
int send_pause_command(struct bt_avctp *session)
{
    uint8_t released = PASTHR_STATE_PRESSED;
    uint8_t opid = AVRCP_KEY_PAUSE;

    int ret = avrcp_pasthr_cmd(session, released, opid);
    if (ret < 0) {
        return ret;
    }

    released = PASTHR_STATE_RELEASED;
    ret = avrcp_pasthr_cmd(session, released, opid);

    return ret;
}

/* 发送音量调节命令 */
int send_volume_command(struct bt_avctp *session, uint8_t is_up)
{
    uint8_t released = PASTHR_STATE_PRESSED;
    uint8_t opid = is_up ? AVRCP_KEY_VOL_UP : AVRCP_KEY_VOL_DOWN;

    int ret = avrcp_pasthr_cmd(session, released, opid);
    if (ret < 0) {
        return ret;
    }

    released = PASTHR_STATE_RELEASED;
    ret = avrcp_pasthr_cmd(session, released, opid);

    return ret;
}
```

### 13.2 获取播放状态

以下示例展示如何查询当前播放状态：

```c
#include "bluetooth/avrcp.h"

/* 获取播放状态 */
int request_play_status(struct bt_avctp *session)
{
    return avrcp_get_play_status_cmd(session);
}

/* 播放状态回调处理 */
void handle_play_status(uint32_t song_len, uint32_t song_pos, uint8_t status)
{
    const char *status_str;

    switch (status) {
        case PLAY_STATUS_STOPPED:
            status_str = "Stopped";
            break;
        case PLAY_STATUS_PLAYING:
            status_str = "Playing";
            break;
        case PLAY_STATUS_PAUSED:
            status_str = "Paused";
            break;
        case PLAY_STATUS_FWD_SEEK:
            status_str = "Fast Forward";
            break;
        case PLAY_STATUS_REV_SEEK:
            status_str = "Fast Rewind";
            break;
        default:
            status_str = "Unknown";
            break;
    }

    printf("Play Status: %s\n", status_str);
    printf("Position: %u ms / %u ms\n", song_pos, song_len);
}
```

### 13.3 事件通知回调注册

以下示例展示如何注册 AVRCP 回调以接收事件通知：

```c
#include "bluetooth/avrcp.h"

/* 定义 AVRCP 回调结构 */
static struct avrcp_callback g_avrcp_cbs = {
    .chain = avrcp_chain_cb,              /* 连接状态回调 */
    .abs_vol = avrcp_abs_vol_cb,         /* 绝对音量回调 */
    .play_status = avrcp_play_status_cb, /* 播放状态回调 */
    .tg_reg_ntf_evt = avrcp_notify_evt_cb, /* 目标事件通知回调 */
    .rp_passthrough = avrcp_passthrough_cb, /* 穿透命令回调 */
};

/* 连接状态回调 */
void avrcp_chain_cb(struct bt_conn *conn, uint8_t state)
{
    if (state == BT_AVRCP_CHAIN_CONNECTED) {
        printf("AVRCP Connected\n");
    } else {
        printf("AVRCP Disconnected\n");
    }
}

/* 绝对音量回调 */
void avrcp_abs_vol_cb(uint8_t vol)
{
    printf("Absolute Volume: %u (max 127)\n", vol);
}

/* 播放状态回调 */
void avrcp_play_status_cb(uint32_t song_len, uint32_t song_pos, uint8_t status)
{
    printf("Song Length: %u ms, Position: %u ms, Status: 0x%02x\n",
           song_len, song_pos, status);
}

/* 事件通知回调 */
void avrcp_notify_evt_cb(uint8_t evt, uint8_t *para, uint16_t para_len)
{
    printf("Event Notification: 0x%02x\n", evt);

    switch (evt) {
        case EVENT_PLAYBACK_STATUS_CHANGED:
            if (para_len >= 1) {
                printf("Playback Status Changed: 0x%02x\n", para[0]);
            }
            break;
        case EVENT_TRACK_CHANGED:
            printf("Track Changed\n");
            break;
        case EVENT_PLAYBACK_POS_CHANGED:
            if (para_len >= 4) {
                uint32_t pos = (para[0] << 24) | (para[1] << 16) |
                              (para[2] << 8) | para[3];
                printf("Playback Position: %u ms\n", pos);
            }
            break;
        case EVENT_VOLUME_CHANGED:
            if (para_len >= 1) {
                printf("Volume Changed: %u\n", para[0] & 0x7F);
            }
            break;
        default:
            printf("Unknown Event\n");
            break;
    }
}

/* 穿透命令回调 */
void avrcp_passthrough_cb(uint8_t released, uint8_t option_id)
{
    const char *state_str = released ? "Released" : "Pressed";
    const char *cmd_str;

    switch (option_id) {
        case AVRCP_KEY_PLAY:
            cmd_str = "Play";
            break;
        case AVRCP_KEY_PAUSE:
            cmd_str = "Pause";
            break;
        case AVRCP_KEY_STOP:
            cmd_str = "Stop";
            break;
        case AVRCP_KEY_VOL_UP:
            cmd_str = "Volume Up";
            break;
        case AVRCP_KEY_VOL_DOWN:
            cmd_str = "Volume Down";
            break;
        default:
            cmd_str = "Unknown";
            break;
    }

    printf("Pass-through: %s - %s\n", cmd_str, state_str);
}

/* 初始化 AVRCP 回调 */
void avrcp_init_callbacks(void)
{
    avrcp_cb_register(&g_avrcp_cbs);
}
```

### 13.4 事件通知注册

以下示例展示如何向目标设备注册事件通知：

```c
#include "bluetooth/avrcp.h"

/* 注册播放状态变化通知 */
int register_playback_status_notification(struct bt_avctp *session)
{
    return avrcp_reg_not_cmd(session, EVENT_PLAYBACK_STATUS_CHANGED);
}

/* 注册播放位置变化通知 */
int register_position_notification(struct bt_avctp *session)
{
    return avrcp_reg_not_cmd(session, EVENT_PLAYBACK_POS_CHANGED);
}

/* 注册音量变化通知 */
int register_volume_notification(struct bt_avctp *session)
{
    return avrcp_reg_not_cmd(session, EVENT_VOLUME_CHANGED);
}

/* 设置播放器参数（用于模拟目标设备） */
void update_player_status(uint8_t status, uint32_t position, uint32_t duration)
{
    avrcp_set_player_parameter(status, position, duration);
}
```

### 13.5 绝对音量控制

以下示例展示如何使用绝对音量控制功能：

```c
#include "bluetooth/avrcp.h"

/* 发送绝对音量设置命令 */
int set_absolute_volume(struct bt_avctp *session, uint8_t volume)
{
    /* 确保音量值在有效范围内 */
    uint8_t avrcp_vol = volume & ABS_VOL_MASK;

    return avrcp_set_absvol_cmd(session, avrcp_vol);
}

/* 发送音量通知（目标设备端） */
int notify_volume_change(struct bt_avctp *session)
{
    return avrcp_send_volume_notification(session);
}
```

## 14. 错误码

AVRCP 定义了以下错误码：

| 错误码 | 值 | 说明 |
|--------|-----|------|
| INVALID_CMD | 0x00 | 无效命令 |
| INVALID_PARAM | 0x01 | 无效参数 |
| PARAM_CONTENT_ERROR | 0x02 | 参数内容错误 |
| INTERNAL_ERROR | 0x03 | 内部错误 |
| OP_COMPLETE_WITHOUT_ERROR | 0x04 | 操作完成无错误 |
| UID_CHANGED | 0x05 | UIDs 已改变 |
| INVALID_DIRECTION | 0x07 | 无效方向 |
| NOT_A_DIRECTORY | 0x08 | 不是目录 |
| DOES_NOT_EXIST | 0x09 | 不存在 |
| INVALID_SCOPE | 0x0A | 无效范围 |
| RANGE_OUT_OF_BOUNDS | 0x0B | 范围越界 |
| FOLDER_ITEM_NOT_PLAYABLE | 0x0C | 文件夹项不可播放 |
| MEDIA_IN_USE | 0x0D | 媒体正在使用中 |
| NOW_PLAYING_LIST_FULL | 0x0E | 当前播放列表已满 |
| SEARCH_NOT_SUPPORTED | 0x0F | 搜索不支持 |
| SEARCH_IN_PROGRESS | 0x10 | 搜索进行中 |
| INVALID_PLAYER_ID | 0x11 | 无效播放器 ID |
| PLAYER_NOT_BROWSABLE | 0x12 | 播放器不可浏览 |
| PLAYER_NOT_ADDRESSED | 0x13 | 播放器未被寻址 |
| NO_VALID_SEARCH_RESULTS | 0x14 | 无有效搜索结果 |
| NO_AVAILABLE_PLAYERS | 0x15 | 没有可用播放器 |
| ADDRESSED_PLAYER_CHANGED | 0x16 | 寻址的播放器已改变 |

## 15. 单元类型

AVRCP 定义了以下单元类型，用于设备信息查询：

| 单元类型 | 值 |
|----------|-----|
| MONITOR | 0x00 |
| AUDIO | 0x01 |
| PRINTER | 0x02 |
| DISC | 0x03 |
| TAPE_RECORDER_PLAYER | 0x04 |
| TUNER | 0x05 |
| CA | 0x06 |
| CAMERA | 0x07 |
| PANEL | 0x09 |
| BULLETIN_BOARD | 0x0A |
| CAMERA_STORAGE | 0x0B |
| VENDOR_UNIQUE | 0x1C |
| SUBUNIT_TYPE_EXTENDED | 0x1E |
| UNIT | 0x1F |

## 16. 初始化流程

### 16.1 AVRCP 初始化

```c
#include "bluetooth/avrcp.h"

/* 初始化 AVRCP */
int avrcp_profile_init(void)
{
    int ret;

    /* 初始化 AVCTP 传输层 */
    ret = bt_avctp_init();
    if (ret < 0) {
        printf("AVCTP init failed: %d\n", ret);
        return ret;
    }

    /* 初始化 AVRCP */
    ret = bt_avrcp_init();
    if (ret < 0) {
        printf("AVRCP init failed: %d\n", ret);
        return ret;
    }

    /* 注册回调 */
    avrcp_init_callbacks();

    return 0;
}
```

### 16.2 建立 AVRCP 连接

```c
#include "bluetooth/avrcp.h"

/* 建立 AVRCP 连接 */
struct bt_avrcp *connect_avrcp(struct bt_conn *conn)
{
    struct bt_avrcp *session;

    session = bt_avrcp_connect(conn);
    if (!session) {
        printf("AVRCP connection failed\n");
        return NULL;
    }

    printf("AVRCP connection initiated\n");
    return session;
}
```

## 参考

- [AVRCP 头文件 - bluetooth/avrcp.h](./bluetooth/avrcp.h)
- [AVCTP 头文件 - bluetooth/avctp.h](./bluetooth/avctp.h)
- Bluetooth SIG: AVRCP Specification
- Bluetooth SIG: AVCTP Specification
