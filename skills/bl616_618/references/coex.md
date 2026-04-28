# Wi-Fi / BLE 共存（Coex）

## 概述

BL616/BL618 支持 Wi-Fi 与 BLE/Thread 同时工作，通过 `coex` 模块实现无线电资源的时间分片调度，避免相互干扰。该模块位于 `components/wireless/coex/`。

## 头文件

```c
#include "coex.h"
```

## 共存模式

```c
#define COEX_MODE_TDMA    1  // 时分复用模式（默认）
#define COEX_MODE_PTI     2  // 优先级时间片模式
```

### TDMA 模式

时分复用模式，将时间划分为固定Slot，Wi-Fi、BLE、Thread 按配置比例分配Tx/Rx时间。

### PTI 模式

优先级时间片模式，通过 `coex_pti.c` 实现，支持优先级仲裁。

## 共存角色

```c
enum coex_role {
    COEX_ROLE_BT = 0,      // 蓝牙
    COEX_ROLE_WIFI,        // Wi-Fi
    COEX_ROLE_THREAD,      // Thread
    COEX_ROLE_DUMMY,
    COEX_ROLE_MAX,
};
```

## 事件类型

```c
enum coex_event {
    COEX_EVT_INIT = 0,           // 初始化无线模块
    COEX_EVT_DEINIT,             // 去初始化无线模块
    COEX_EVT_SET_ACTIVITY,       // 设置无线模块活动
    COEX_EVT_GET_ACTIVITY,       // 获取共存模块活动
    COEX_EVT_TMR_ISR_HANDLE,     // 定时器中断处理
    COEX_EVT_FUNC_CALL,          // 在共存模块中调用函数
    COEX_EVT_MAX,
};
```

## 活动类型（Activity）

Wi-Fi、BLE、Thread 各自的活动事件，用于通知共存调度器：

```c
enum coex_event_activity {
    /* BLE */
    ACT_START_ADV,           // BLE 广播开始

    /* BT Classic */
    ACT_BT_SCAN_START,       // BT 扫描开始
    ACT_BT_SCAN_DONE,        // BT 扫描完成
    ACT_BT_CONNECT_START,    // BT 连接开始
    ACT_BT_CONNECT_DONE_OK,  // BT 连接成功
    ACT_BT_CONNECT_DONE_FAIL,// BT 连接失败
    ACT_BT_DISCONNECT_START, // BT 断开开始
    ACT_BT_DISCONNECT_DONE,  // BT 断开完成
    ACT_BT_A2DP_START,      // A2DP 音频开始
    ACT_BT_A2DP_STOP,       // A2DP 音频停止

    /* Wi-Fi STA */
    ACT_STA_SCAN_START,       // Wi-Fi 扫描开始
    ACT_STA_SCAN_DONE,       // Wi-Fi 扫描完成
    ACT_STA_CONNECT_START,   // Wi-Fi 连接开始
    ACT_STA_CONNECT_DONE_OK, // Wi-Fi 连接成功
    ACT_STA_CONNECT_DONE_FAIL,// Wi-Fi 连接失败
    ACT_STA_DISCONNECT_START,// Wi-Fi 断开开始
    ACT_STA_DISCONNECT_DONE, // Wi-Fi 断开完成
    ACT_STA_DPSM_START,      // 功耗状态机开始
    ACT_STA_DPSM_YIELD,     // 功耗状态机让出
    ACT_STA_DPSM_STOP,      // 功耗状态机停止
    ACT_STA_ROC_REQ,        // Remain-on-Channel 请求
    ACT_STA_TBTT_UPDATE,    // TBTT 更新（Beacon 同步）

    /* Wi-Fi AP */
    ACT_SOFTAP_START,        // SoftAP 启动
    ACT_SOFTAP_STOP,         // SoftAP 停止
    ACT_SOFTAP_TBTT_UPDATE,  // AP TBTT 更新

    /* Thread */
    ACT_START_PAN,           // PAN 启动
    ACT_STOP_PAN,            // PAN 停止

    /* Dummy */
    ACT_DUMMY_ADD_ACT,       // 虚拟活动添加
    ACT_DUMMY_DEL_ACT,       // 虚拟活动删除
};
```

## 回调通知

共存模块通过回调通知上层事件：

```c
struct coex_notify_args {
    int event;      // @COEX_NTY_* 事件码
    int duration;   // 持续时间（ms）
};

typedef void (*coex_notify_cb)(void *env, struct coex_notify_args *args);
```

## 通知事件

```c
enum coex_notify {
    COEX_NTY_INITED = 0,      // 共存已初始化
    COEX_NTY_DEINITED,        // 共存已去初始化
    COEX_NTY_RF_PRESENCE,     // 射频正在使用
    COEX_NTY_RF_ABSENCE,      // 射频空闲
    COEX_NTY_MAX,
};
```

## 事件参数联合体

```c
union evt_arg {
    struct {              // INIT 事件
        coex_notify_cb cb;
        void *env;
    } init;
    struct {              // SET_ACTIVITY 事件
        int type;         // activity 类型 @ coex_event_activity
        int now;          // 发生时间
    } set_act;
    struct {              // TMR_ISR 事件
        uint64_t time;
        void *env;
    } tmr_isr;
    struct {              // FUNC_CALL 事件
        coex_func_call func;
        int arg[4];
    } func_call;
};

struct coex_evt_arg {
    int role;      // @ coex_role
    int type;      // @ coex_event
    union evt_arg arg;
};
```

## 核心 API

### 初始化 / 去初始化

```c
int coex_init(void);    // 返回 COEX_OK (0) 或 COEX_FAIL (-1)
int coex_deinit(void);
```

### 事件通知

```c
int coex_event(struct coex_evt_arg *arg);
```

向共存模块报告无线模块事件（Wi-Fi 扫描/BT 连接等），共存调度器据此调整时间片分配。

## 工作代码示例

### 基本初始化

```c
#include "coex.h"

static void coex_notify(void *env, struct coex_notify_args *args)
{
    switch (args->event) {
    case COEX_NTY_INITED:
        printf("Coex initialized\r\n");
        break;
    case COEX_NTY_RF_PRESENCE:
        printf("RF active, duration=%dms\r\n", args->duration);
        break;
    case COEX_NTY_RF_ABSENCE:
        printf("RF idle\r\n");
        break;
    }
}

void coex_example(void)
{
    struct coex_evt_arg evt;
    int ret;

    /* 初始化共存模块 */
    ret = coex_init();
    if (ret != COEX_OK) {
        printf("coex init failed\r\n");
        return;
    }

    /* 注册通知回调 */
    evt.role = COEX_ROLE_WIFI;
    evt.type = COEX_EVT_INIT;
    evt.arg.init.cb = coex_notify;
    evt.arg.init.env = NULL;
    coex_event(&evt);

    /* ... Wi-Fi 和 BLE 业务逻辑 ... */

    coex_deinit();
}
```

### Wi-Fi 扫描时通知共存

```c
void wifi_scan_with_coex(void)
{
    struct coex_evt_arg evt;

    /* 通知 BLE：Wi-Fi 即将扫描 */
    evt.role = COEX_ROLE_WIFI;
    evt.type = COEX_EVT_SET_ACTIVITY;
    evt.arg.set_act.type = ACT_STA_SCAN_START;
    evt.arg.set_act.now = 1;
    coex_event(&evt);

    /* 执行 Wi-Fi 扫描 */
    wifi_mgmr_sta_scan(NULL);

    /* 通知 BLE：Wi-Fi 扫描完成 */
    evt.arg.set_act.type = ACT_STA_SCAN_DONE;
    coex_event(&evt);
}
```

### BLE 连接时通知共存

```c
void ble_connect_with_coex(void)
{
    struct coex_evt_arg evt;

    /* 通知 Wi-Fi：BLE 即将连接 */
    evt.role = COEX_ROLE_BT;
    evt.type = COEX_EVT_SET_ACTIVITY;
    evt.arg.set_act.type = ACT_BT_CONNECT_START;
    evt.arg.set_act.now = 1;
    coex_event(&evt);

    /* 执行 BLE 连接 */
    ble_gap_connect(...);

    /* 连接成功后通知 */
    evt.arg.set_act.type = ACT_BT_CONNECT_DONE_OK;
    coex_event(&evt);
}
```

### A2DP 音频播放时优先级处理

```c
void a2dp_playback_with_coex(void)
{
    struct coex_evt_arg evt;

    /* A2DP 开始 — 分配较高优先级 */
    evt.role = COEX_ROLE_BT;
    evt.type = COEX_EVT_SET_ACTIVITY;
    evt.arg.set_act.type = ACT_BT_A2DP_START;
    evt.arg.set_act.now = 1;
    coex_event(&evt);

    /* A2DP 流媒体传输期间，Wi-Fi 扫描会降低优先级 */

    /* A2DP 停止 */
    evt.arg.set_act.type = ACT_BT_A2DP_STOP;
    coex_event(&evt);
}
```

## 配置宏

通过 Kconfig 配置共存模式：

| 宏 | 默认值 | 说明 |
|----|--------|------|
| `CONFIG_COEX_WIFI_MODE` | `0` | Wi-Fi 共存模式（0=关闭，1=TDMA，2=PTI） |
| `CONFIG_COEX_THREAD_MODE` | `0` | Thread 共存模式 |
| `CONFIG_COEX_BT_MODE` | `0` | BT/BLE 共存模式 |
| `CONFIG_COEX_TDMA_NONE` | `COEX_NONE_NULL` | 无活动时行为 |

## 共存调度策略

1. **时间分片**：TDMA 模式下，调度器将时间划分为固定Slot，比例可在配置文件中指定
2. **优先级仲裁**：PTI 模式下，高优先级活动（如 A2DP）可抢占低优先级活动（如 Wi-Fi 扫描）的时间片
3. **活动通报**：Wi-Fi/BLE/Thread 驱动通过 `coex_event()` 报告当前活动，调度器据此动态调整
4. **射频状态**：通过 `COEX_NTY_RF_PRESENCE/_ABSENCE` 通知上层，应用程序可据此决定是否发起新活动

## 相关文档

- [wifi_mgmr](./wifi_mgmr.md) — Wi-Fi 管理器（内部调用 coex）
- [BLE](./ble.md) — BLE 控制器
- [bt_a2dp](./bt_a2dp.md) — A2DP 音频配置
