# Power Management (PM) API 参考

> 来源文件：`components/network/wifi_hosal/include/wifi_hosal.h`（部分）  
> BL602 的电源管理涉及 RF 射频、Wi-Fi 休眠、PDS（Power Down Sleep）等多种低功耗机制。

---

## 概述

BL602 电源管理架构：

```
┌──────────────────────────────────────┐
│           Application               │
├──────────────────────────────────────┤
│     Wi-Fi / BLE / System PM         │
├──────────────────────────────────────┤
│   RF (Radio Frequency)  射频         │
│   PDS (Power Down Sleep) 深度休眠    │
│   Normal Run  正常运行               │
└──────────────────────────────────────┘
```

**低功耗级别**：

| 级别 | 模式 | 功耗 | 说明 |
|------|------|------|------|
| `PM_LEVEL_NONE` | 全速运行 | 最高 | Wi-Fi 传输中 |
| `PM_LEVEL_1` | 浅睡眠 | 低 | RF 保持连接 |
| `PM_LEVEL_2` | 深度睡眠 | 极低 | RAM 保留，仅唤醒源 |
| `PM_MODE_MAX` | 关闭 | 最低 | 需重新初始化 |

---

## Wi-Fi HOSAL PM 函数

以下函数来自 `wifi_hosal.h`：

### `wifi_hosal_pm_init`

初始化电源管理。

```c
int wifi_hosal_pm_init(void);
```

---

### `wifi_hosal_pm_event_register`

注册 PM 事件回调。

```c
int wifi_hosal_pm_event_register(enum PM_EVEMT event,
                                   uint32_t code,
                                   uint32_t cap_bit,
                                   uint16_t priority,
                                   bl_pm_cb_t ops,
                                   void *arg,
                                   enum PM_EVENT_ABLE enable);
```

---

### `wifi_hosal_pm_deinit`

关闭电源管理。

```c
int wifi_hosal_pm_deinit(void);
```

---

### `wifi_hosal_pm_state_run`

进入运行状态（退出低功耗）。

```c
int wifi_hosal_pm_state_run(void);
```

---

### `wifi_hosal_pm_capacity_set`

设置低功耗级别。

```c
int wifi_hosal_pm_capacity_set(enum PM_LEVEL level);
```

| `level` | 说明 |
|---------|------|
| `PM_LEVEL_NONE` | 退出低功耗 |
| `PM_LEVEL_1` | 浅睡眠 |
| `PM_LEVEL_2` | 深度睡眠 |

---

### `wifi_hosal_pm_post_event`

投递 PM 事件。

```c
int wifi_hosal_pm_post_event(enum PM_EVEMT event, uint32_t code, uint32_t *retval);
```

---

### `wifi_hosal_pm_event_switch`

使能/禁用 PM 事件。

```c
int wifi_hosal_pm_event_switch(enum PM_EVEMT event, uint32_t code,
                                 enum PM_EVENT_ABLE enable);
```

---

## RF 电源控制

### `wifi_hosal_rf_turn_on`

开启 RF 射频。

```c
int wifi_hosal_rf_turn_on(void *arg);
```

---

### `wifi_hosal_rf_turn_off`

关闭 RF 射频。

```c
int wifi_hosal_rf_turn_off(void *arg);
```

---

## Wi-Fi MGMR 功耗接口

以下函数来自 `wifi_mgmr_ext.h`：

### `wifi_mgmr_sta_ps_enter`

Wi-Fi STA 进入低功耗。

```c
int wifi_mgmr_sta_ps_enter(uint32_t ps_level);
```

> 参数为 `PS_MODE_OFF`（关闭）、`PS_MODE_ON`（普通）、`PS_MODE_ON_DYN`（动态）

---

### `wifi_mgmr_sta_ps_exit`

Wi-Fi STA 退出低功耗。

```c
int wifi_mgmr_sta_ps_exit(void);
```

---

### `wifi_mgmr_set_wifi_active_time`

设置 Wi-Fi 活跃时间。

```c
int wifi_mgmr_set_wifi_active_time(uint32_t ms);
```

---

### `wifi_mgmr_set_listen_interval`

设置监听间隔（ beacon 数量）。

```c
int wifi_mgmr_set_listen_interval(uint16_t itv);
```

---

## 使用示例

```c
#include "wifi_hosal.h"

// 初始化 PM
wifi_hosal_pm_init();

// 设置低功耗级别
wifi_hosal_pm_capacity_set(PM_LEVEL_1);

// Wi-Fi 低功耗配置
wifi_mgmr_set_wifi_active_time(100);    // 活跃 100ms
wifi_mgmr_set_listen_interval(10);      // 每 10 个 beacon 唤醒一次
wifi_mgmr_sta_ps_enter(PS_MODE_ON_DYN); // 进入动态低功耗模式

// 退出低功耗
wifi_mgmr_sta_ps_exit();
wifi_hosal_pm_state_run();

// 关闭 RF（极度省电，仅限特定场景）
wifi_hosal_rf_turn_off(NULL);

// 重新开启 RF
wifi_hosal_rf_turn_on(NULL);
```
