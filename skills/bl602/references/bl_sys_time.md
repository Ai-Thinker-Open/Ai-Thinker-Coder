# 系统时间 API 参考

> 来源文件：`components/sys/bltime/include/bl_sys_time.h`  
> 系统时间管理，基于 RTC 硬件，支持 NTP 同步和时间获取。

---

## 概述

`bl_sys_time` 提供系统级时间管理，基于 RTC 实现，支持：
- 获取当前 Epoch 时间（秒）
- 更新系统时间（从 NTP 或其他时间源）
- NTP 自动同步

---

## 头文件

```c
#include "bl_sys_time.h"
```

---

## 函数接口

### `bl_sys_time_get`

获取当前系统时间（Epoch 秒）。

```c
int bl_sys_time_get(uint64_t *epoch);
```

| 参数 | 说明 |
|------|------|
| `epoch` | 输出的 Epoch 时间（秒） |

**返回值**：0=成功，-1=失败

---

### `bl_sys_time_update`

手动更新系统时间。

```c
void bl_sys_time_update(uint64_t epoch);
```

| 参数 | 说明 |
|------|------|
| `epoch` | 新的 Epoch 时间（秒） |

---

### `bl_sys_time_cli_init`

初始化时间 CLI 命令（可通过 CLI 设置/查询时间）。

```c
int bl_sys_time_cli_init(void);
```

---

### `bl_sys_time_sync_init`

初始化 NTP 自动同步。

```c
void bl_sys_time_sync_init(void);
```

---

### `bl_sys_time_sync`

手动触发一次 NTP 同步。

```c
uint32_t bl_sys_time_sync(void);
```

**返回值**：同步后的系统时间（Epoch 秒）

---

### `bl_sys_time_sync_state`

获取时间同步状态。

```c
int bl_sys_time_sync_state(uint32_t *xTicksToJump);
```

| 参数 | 说明 |
|------|------|
| `xTicksToJump` | 输出上次的跳跃 Tick 数 |

**返回值**：同步状态

---

## 使用示例

### 基本时间获取

```c
#include "bl_sys_time.h"

void print_current_time(void)
{
    uint64_t epoch;
    if (bl_sys_time_get(&epoch) == 0) {
        printf("Epoch: %llu\r\n", epoch);
        // 转换为可读格式
        time_t t = (time_t)epoch;
        struct tm *tm_info = localtime(&t);
        printf("Time: %s", asctime(tm_info));
    }
}
```

### 初始化 NTP 同步

```c
void time_init(void)
{
    // 初始化 NTP 自动同步
    bl_sys_time_sync_init();

    // 等待首次同步
    vTaskDelay(pdMS_TO_TICKS(3000));

    uint64_t now;
    bl_sys_time_get(&now);
    printf("Time synchronized: %llu\r\n", now);
}
```
