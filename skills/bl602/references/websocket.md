# WebSocket API 参考

> 来源文件：`components/network/axk_protocol_stack/tcp_transport/include/axk_transport_ws.h`  
> WebSocket 协议封装，基于 TCP Transport，支持文本/二进制帧、Ping/Pong、握手升级。

---

## 概述

WebSocket 协议建立在 HTTP 协议之上，通过 HTTP Upgrade 机制切换到 WebSocket 连接。BL602 支持：
- WS（WebSocket 明文）
- WSS（WebSocket over TLS）
- 文本/二进制帧收发
- Ping/Pong 心跳
- 自定义 HTTP Header（User-Agent、Sub-Protocol）
- 连接关闭检测

---

## 头文件

```c
#include "axk_transport_ws.h"
```

---

## 帧类型

```c
typedef enum ws_transport_opcodes {
    WS_TRANSPORT_OPCODES_CONT =  0x00,   // 延续帧
    WS_TRANSPORT_OPCODES_TEXT =  0x01,  // 文本帧
    WS_TRANSPORT_OPCODES_BINARY = 0x02, // 二进制帧
    WS_TRANSPORT_OPCODES_CLOSE = 0x08,   // 关闭帧
    WS_TRANSPORT_OPCODES_PING = 0x09,    // Ping 帧
    WS_TRANSPORT_OPCODES_PONG = 0x0a,    // Pong 帧
    WS_TRANSPORT_OPCODES_FIN = 0x80,    // FIN 标志
    WS_TRANSPORT_OPCODES_NONE = 0x100,   // 无效 opcode
} ws_transport_opcodes_t;
```

---

## 配置结构

### `axk_transport_ws_config_t`

```c
typedef struct {
    const char *ws_path;                    // WebSocket 路径（如 "/ws"）
    const char *sub_protocol;              // 子协议（如 "mqtt"）
    const char *user_agent;                // User-Agent 头
    const char *headers;                   // 额外 HTTP 头（每行以 \r\n 结尾）
    bool propagate_control_frames;         // true=控制帧传递给读取方，false=自动处理
} axk_transport_ws_config_t;
```

---

## 函数接口

### `axk_transport_ws_init`

创建 WebSocket 传输层。

```c
axk_transport_handle_t axk_transport_ws_init(axk_transport_handle_t parent_handle);
```

| 参数 | 说明 |
|------|------|
| `parent_handle` | 父级 TCP/SSL 传输句柄 |

**返回值**：WebSocket 传输句柄

---

### `axk_transport_ws_set_path`

设置 WebSocket 路径。

```c
void axk_transport_ws_set_path(axk_transport_handle_t t, const char *path);
```

---

### `axk_transport_ws_set_subprotocol`

设置子协议。

```c
axk_err_t axk_transport_ws_set_subprotocol(axk_transport_handle_t t,
                                            const char *sub_protocol);
```

---

### `axk_transport_ws_set_user_agent`

设置 User-Agent 头。

```c
axk_err_t axk_transport_ws_set_user_agent(axk_transport_handle_t t,
                                            const char *user_agent);
```

---

### `axk_transport_ws_set_headers`

设置额外 HTTP 头。

```c
axk_err_t axk_transport_ws_set_headers(axk_transport_handle_t t,
                                         const char *headers);
```

---

### `axk_transport_ws_set_config`

批量设置 WebSocket 配置。

```c
axk_err_t axk_transport_ws_set_config(axk_transport_handle_t t,
                                        const axk_transport_ws_config_t *config);
```

---

### `axk_transport_ws_send_raw`

发送自定义 opcode 的原始帧。

```c
int axk_transport_ws_send_raw(axk_transport_handle_t t,
                                ws_transport_opcodes_t opcode,
                                const char *buffer,
                                int len,
                                int timeout_ms);
```

| 参数 | 说明 |
|------|------|
| `opcode` | 帧类型（TEXT/BINARY/PING/PONG/CLOSE） |
| `buffer` | 数据缓冲区 |
| `len` | 数据长度（0=发送 Ping） |
| `timeout_ms` | 超时（毫秒，-1=永久阻塞） |

---

### `axk_transport_ws_get_fin_flag`

获取最后收到帧的 FIN 标志。

```c
bool axk_transport_ws_get_fin_flag(axk_transport_handle_t t);
```

---

### `axk_transport_ws_get_read_opcode`

获取最后收到帧的 opcode。

```c
ws_transport_opcodes_t axk_transport_ws_get_read_opcode(axk_transport_handle_t t);
```

---

### `axk_transport_ws_get_read_payload_len`

获取最后收到帧的载荷长度。

```c
int axk_transport_ws_get_read_payload_len(axk_transport_handle_t t);
```

---

### `axk_transport_ws_poll_connection_closed`

等待连接关闭事件。

```c
int axk_transport_ws_poll_connection_closed(axk_transport_handle_t t,
                                             int timeout_ms);
```

| 返回值 | 说明 |
|--------|------|
| 0 | 超时 |
| 1 | 连接已 FIN 关闭或收到 RST |
| -1 | 失败 |

---

## 使用示例

### WebSocket 客户端

```c
#include "axk_transport_ws.h"

static void ws_task(void *arg)
{
    // 创建 TCP 传输
    axk_transport_handle_t parent = axk_transport_tcp_init();

    // 创建 WebSocket 传输
    axk_transport_handle_t ws = axk_transport_ws_init(parent);

    // 设置 WebSocket 配置
    axk_transport_ws_set_path(ws, "/echo");
    axk_transport_ws_set_subprotocol(ws, "echo");

    // 连接到服务器
    axk_transport_connect(ws, "echo.websocket.org", 80, 5000);

    // 发送文本消息
    const char *msg = "Hello WebSocket";
    axk_transport_ws_send_raw(ws, WS_TRANSPORT_OPCODES_TEXT,
                              msg, strlen(msg), 5000);

    // 读取响应
    char buf[512];
    int len = axk_transport_read(ws, buf, sizeof(buf), 5000);
    if (len > 0) {
        printf("Received: %.*s\r\n", len, buf);

        // 检查 opcode
        ws_transport_opcodes_t opcode = axk_transport_ws_get_read_opcode(ws);
        if (opcode == WS_TRANSPORT_OPCODES_TEXT) {
            printf("Text frame\r\n");
        } else if (opcode == WS_TRANSPORT_OPCODES_BINARY) {
            printf("Binary frame\r\n");
        }
    }

    // 发送 Ping（len=0 发送 Ping 帧）
    axk_transport_ws_send_raw(ws, WS_TRANSPORT_OPCODES_PING, NULL, 0, 5000);

    // 关闭连接
    axk_transport_ws_send_raw(ws, WS_TRANSPORT_OPCODES_CLOSE, NULL, 0, 5000);
    axk_transport_close(ws);

    vTaskDelete(NULL);
}
```

### 带 TLS 的 WebSocket（WSS）

```c
axk_transport_handle_t ssl = axk_transport_ssl_init();
axk_transport_handle_t ws = axk_transport_ws_init(ssl);

axk_transport_ssl_set_cert_data(ssl, ca_cert_pem, strlen(ca_cert_pem) + 1);
axk_transport_ws_set_path(ws, "/wss");

axk_transport_connect(ws, "secure-websocket.example.com", 443, 10000);
```

---

## 与 MQTT over WebSocket 的区别

| 特性 | WebSocket | MQTT over WS |
|------|-----------|--------------|
| 用途 | 通用双向通信 | MQTT 协议承载 |
| 子协议 | 可选自定义 | `mqtt` |
| 帧类型 | TEXT/BINARY | BINARY |
| 库 | axk_transport_ws | mqtt_client (transport=WS/WSS) |
