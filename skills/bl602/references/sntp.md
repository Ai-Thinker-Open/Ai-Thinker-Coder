# SNTP 时间同步 API 参考

> 来源文件：`components/network/sntp/include/sntp.h`  
> 基于标准 lwIP SNTP 客户端，支持 NTP 服务器轮询和回调通知。

---

## 概述

SNTP 客户端用于从 NTP 服务器同步系统时间，支持：
- 单播/广播模式
- 多服务器配置
- 可达性监控
- 同步回调通知

---

## 头文件

```c
#include "sntp.h"
```

---

## 类型定义

### `ntp_sync_cb`

时间同步回调函数类型：

```c
typedef void (*ntp_sync_cb)(void);
```

---

## 函数接口

### `sntp_setoperatingmode`

设置 SNTP 工作模式（需在 `sntp_init` 前调用）。

```c
void sntp_setoperatingmode(u8_t operating_mode);
```

| 模式 | 值 | 说明 |
|------|----|------|
| `SNTP_OPMODE_POLL` | 0 | 轮询模式（默认） |
| `SNTP_OPMODE_LISTENONLY` | 1 | 仅监听广播 |

---

### `sntp_getoperatingmode`

获取当前工作模式：

```c
u8_t sntp_getoperatingmode(void);
```

---

### `sntp_init`

初始化并启动 SNTP 客户端。

```c
void sntp_init(void);
```

---

### `sntp_stop`

停止 SNTP 客户端。

```c
void sntp_stop(void);
```

---

### `sntp_enabled`

查询 SNTP 是否已启用。

```c
u8_t sntp_enabled(void);
```

**返回值**：1=已启用，0=未启用

---

### `sntp_setserver`

设置 NTP 服务器地址（按索引）。

```c
void sntp_setserver(u8_t idx, const ip_addr_t *addr);
```

| 参数 | 说明 |
|------|------|
| `idx` | 服务器索引（0~3） |
| `addr` | IP 地址 |

---

### `sntp_getserver`

获取 NTP 服务器地址。

```c
const ip_addr_t *sntp_getserver(u8_t idx);
```

---

### `sntp_setservername`

通过域名设置 NTP 服务器（需 `SNTP_SERVER_DNS=1`）。

```c
void sntp_setservername(u8_t idx, const char *server);
```

---

### `sntp_getservername`

获取 NTP 服务器域名。

```c
const char *sntp_getservername(u8_t idx);
```

---

### `sntp_getreachability`

获取服务器可达性状态（需 `SNTP_MONITOR_SERVER_REACHABILITY=1`）。

```c
u8_t sntp_getreachability(u8_t idx);
```

**返回值**：0=不可达，非0=可达

---

### `sntp_get_time`

获取当前时间（高分辨率）。

```c
int sntp_get_time(uint32_t *seconds, uint32_t *frags);
```

| 参数 | 说明 |
|------|------|
| `seconds` | Epoch 秒数（输出） |
| `frags` | 分数秒（输出） |

**返回值**：0=成功

---

### `sntp_settimesynccb`

设置时间同步回调（同步成功时自动调用）。

```c
void sntp_settimesynccb(ntp_sync_cb cb);
```

---

### `sntp_setupdatedelay`

设置同步间隔。

```c
void sntp_setupdatedelay(uint32_t delay);
```

---

### `sntp_cli_init`

初始化 SNTP CLI 命令（可通过 CLI 操作）。

```c
int sntp_cli_init(void);
```

---

## 使用示例

### 基本初始化

```c
#include "sntp.h"
#include "lwip/apps/sntp_opts.h"

void sntp_example(void)
{
    // 设置工作模式
    sntp_setoperatingmode(SNTP_OPMODE_POLL);

    // 设置 NTP 服务器（可用域名）
    sntp_setservername(0, "pool.ntp.org");
    sntp_setservername(1, "time.google.com");

    // 初始化
    sntp_init();
}
```

### 带回调的初始化

```c
static void on_time_synced(void)
{
    uint32_t sec, frags;
    sntp_get_time(&sec, &frags);
    printf("Time synced: %u.%u\r\n", sec, frags);
}

void sntp_with_callback(void)
{
    sntp_setoperatingmode(SNTP_OPMODE_POLL);
    sntp_setservername(0, "pool.ntp.org");
    sntp_settimesynccb(on_time_synced);
    sntp_init();
}
```
