# Ping (ICMP) API 参考

> 来源文件：`components/network/netutils/include/ping.h`  
> lwIP 内置 ICMP ping 工具，用于网络诊断。

---

## 概述

Ping 工具基于 lwIP 的 ICMP 实现，可用于检测设备与目标主机的网络连通性和往返延迟（RTT）。

---

## 头文件

```c
#include "ping.h"
```

---

## 类型定义

### `ping_option`

Ping 选项配置：

```c
typedef struct {
    uint32_t count;        // Ping 次数（0=无限）
    uint32_t interval;     // 间隔（秒）
    uint32_t timeout;      // 超时（秒）
    uint32_t data_size;   // 数据载荷大小（字节）
    ip_addr_t target;      // 目标 IP 地址
} ping_option_t;
```

---

### `ping_result`

Ping 结果：

```c
typedef struct {
    uint32_t total_count;       // 总发送数
    uint32_t total_success;     // 成功响应数
    uint32_t total_fail;        // 失败数
    uint32_t avg_time;          // 平均 RTT（ms）
    uint32_t min_time;          // 最小 RTT（ms）
    uint32_t max_time;          // 最大 RTT（ms）
} ping_result_t;
```

---

## 函数接口

### `ping_init`

初始化 Ping 模块。

```c
int ping_init(void);
```

---

### `ping_send`

发送一次 Ping 请求。

```c
int ping_send(const char *host);
```

| 参数 | 说明 |
|------|------|
| `host` | 目标主机（域名或 IP） |

**返回值**：0=发送成功

---

### `ping_raw_send`

发送原始 ICMP Ping（需手动设置目标 IP）。

```c
int ping_raw_send(ip_addr_t *addr);
```

---

### `ping_set_option`

设置 Ping 选项。

```c
int ping_set_option(ping_option_t *option);
```

---

### `ping_get_result`

获取 Ping 统计结果。

```c
int ping_get_result(ping_result_t *result);
```

---

### `ping_register_result_callback`

注册结果回调（每次收到响应时调用）。

```c
int ping_register_result_callback(void (*callback)(void *arg));
```

---

## 使用示例

### 简单 Ping

```c
#include "ping.h"

void ping_test(void)
{
    ping_init();

    // Ping 3次
    for (int i = 0; i < 3; i++) {
        int ret = ping_send("192.168.1.1");
        if (ret == 0) {
            printf("Ping %d: OK\r\n", i + 1);
        } else {
            printf("Ping %d: Failed\r\n", i + 1);
        }
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 带统计的 Ping

```c
void ping_with_stats(const char *target)
{
    ping_init();

    ping_option_t opt = {
        .count = 5,
        .interval = 1,
        .timeout = 3,
        .data_size = 32,
    };
    ping_set_option(&opt);

    ping_send(target);

    // 等待所有响应
    vTaskDelay(pdMS_TO_TICKS(6000));

    ping_result_t result;
    ping_get_result(&result);

    printf("Ping stats:\r\n");
    printf("  Sent: %u, Success: %u, Fail: %u\r\n",
           result.total_count, result.total_success, result.total_fail);
    printf("  RTT: avg=%ums min=%ums max=%ums\r\n",
           result.avg_time, result.min_time, result.max_time);
}
```
