# HTTPS API 参考（简明版）

> 来源文件：`components/network/https/include/https.h`  
> BL602 简化的 HTTPS 客户端接口，基于 TLS 连接提供加密 TCP 数据收发。

---

## 概述

`https.h` 提供了一套简化版 HTTPS 接口，内部封装了 TLS 连接过程，适合快速实现 HTTPS 请求场景。相比完整的 axk_tls API，本接口更易用但功能有限。

**注意**：对于复杂 HTTPS 场景（如自定义 HTTP 头、证书验证），建议使用 `axk_tls.h` 或 `http_client.h`。

---

## 头文件

```c
#include "https.h"
```

---

## 函数接口

### `blTcpSslConnect`

建立加密 TCP 连接。

```c
int32_t blTcpSslConnect(const char *dst, uint16_t port);
```

| 参数 | 说明 |
|------|------|
| `dst` | 目标服务器地址（域名或 IP） |
| `port` | 端口号 |

| 返回值 | 说明 |
|--------|------|
| >0 | 成功，返回 socket 描述符 |
| `BL_TCP_CREATE_CONNECT_ERR` | 连接创建失败 |
| `BL_TCP_ARG_INVALID` | 参数无效（dst 为空） |

> 返回 socket 后需通过 `blTcpSslState()` 判断 TLS 握手是否完成

---

### `blTcpSslState`

查询加密连接状态。

```c
int32_t blTcpSslState(int32_t fd);
```

| 返回值 | 说明 |
|--------|------|
| `BL_TCP_STATE_CONNECTED` | TLS 握手完成，可收发数据 |
| `BL_TCP_STATE_CONNECTING` | 握手进行中 |
| `BL_TCP_STATE_FAILED` | 连接失败 |
| 其他 | 错误码 |

---

### `blTcpSslDisconnect`

断开加密连接。

```c
void blTcpSslDisconnect(int32_t fd);
```

---

### `blTcpSslSend`

发送加密数据（非阻塞）。

```c
int32_t blTcpSslSend(int32_t fd, const uint8_t *buf, uint16_t len);
```

| 参数 | 说明 |
|------|------|
| `fd` | `blTcpSslConnect` 返回的 socket |
| `buf` | 发送数据缓冲区 |
| `len` | 数据长度（范围 0~512） |

| 返回值 | 说明 |
|--------|------|
| >=0 | 实际发送字节数 |
| 错误码 | 发送失败 |

---

### `blTcpSslRead`

读取加密数据（非阻塞）。

```c
int32_t blTcpSslRead(int32_t fd, uint8_t *buf, uint16_t len);
```

| 参数 | 说明 |
|------|------|
| `fd` | `blTcpSslConnect` 返回的 socket |
| `buf` | 接收数据缓冲区 |
| `len` | 缓冲区最大长度（范围 0~512） |

| 返回值 | 说明 |
|--------|------|
| >=0 | 实际读取字节数 |
| 错误码 | 读取失败 |

---

## 使用示例

### 简单 HTTPS GET

```c
#include "https.h"

void https_get_example(void)
{
    // 建立 HTTPS 连接
    int32_t fd = blTcpSslConnect("www.example.com", 443);
    if (fd < 0) {
        printf("Connect failed: %ld\r\n", fd);
        return;
    }

    // 等待 TLS 握手完成
    while (blTcpSslState(fd) == BL_TCP_STATE_CONNECTING) {
        vTaskDelay(pdMS_TO_TICKS(10));
    }

    // 发送 HTTP 请求
    const char *request =
        "GET / HTTP/1.1\r\n"
        "Host: www.example.com\r\n"
        "User-Agent: BL602\r\n"
        "Connection: close\r\n"
        "\r\n";

    int32_t sent = blTcpSslSend(fd, (const uint8_t *)request,
                                  strlen(request));
    if (sent < 0) {
        printf("Send failed\r\n");
        blTcpSslDisconnect(fd);
        return;
    }

    // 读取响应
    uint8_t buf[512];
    int32_t len;
    while ((len = blTcpSslRead(fd, buf, sizeof(buf))) > 0) {
        printf("%.*s", (int)len, buf);
    }

    blTcpSslDisconnect(fd);
}
```

### 带超时检测

```c
int32_t wait_for_ssl_connected(int32_t fd, uint32_t timeout_ms)
{
    uint32_t start = xTaskGetTickCount();
    while (blTcpSslState(fd) == BL_TCP_STATE_CONNECTING) {
        if ((xTaskGetTickCount() - start) * portTICK_PERIOD_MS > timeout_ms) {
            return -1; // 超时
        }
        vTaskDelay(pdMS_TO_TICKS(10));
    }
    return 0;
}
```

---

## 错误码参考

| 错误码 | 说明 |
|--------|------|
| `BL_TCP_ARG_INVALID` | 参数无效 |
| `BL_TCP_CREATE_CONNECT_ERR` | 创建连接失败 |
| `BL_TCP_STATE_CONNECTED` | 已连接 |
| `BL_TCP_STATE_CONNECTING` | 连接中 |
| `BL_TCP_STATE_FAILED` | 连接失败 |

---

## 与 axk_tls 的区别

| 特性 | `https.h` | `axk_tls.h` |
|------|-----------|-------------|
| API 复杂度 | 简单 | 复杂 |
| 非阻塞模式 | 支持 | 支持 |
| 证书配置 | 有限 | 完整（CA/客户端证书/PSK） |
| 错误处理 | 简化 | 详细 |
| 适用场景 | 简单 HTTPS 请求 | 双向认证、PSK 等 |
