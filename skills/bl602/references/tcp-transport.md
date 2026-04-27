# TCP Transport API 参考

> 来源文件：`components/network/axk_protocol_stack/tcp_transport/include/axk_transport.h`  
> 传输层抽象，统一管理 TCP/SSL/WS/WSS 多种传输方式。

---

## 概述

TCP Transport 是 BL602 网络协议栈的传输层抽象，提供了统一的接口来管理 TCP、SSL、WebSocket 等传输方式。它是 MQTT、HTTP 等高层协议的基础。

```
应用层 (MQTT / HTTP / 自定义协议)
         ↓
  TCP Transport (统一接口)
         ↓
  ┌─────────┬──────────┬─────────┬──────────┐
  ↓         ↓          ↓         ↓          ↓
 TCP     SSL        WS        WSS      其他传输
```

---

## 头文件

```c
#include "axk_transport.h"
```

---

## 类型定义

### Keep-Alive 配置

```c
typedef struct axk_transport_keepalive {
    bool keep_alive_enable;      // 使能
    int  keep_alive_idle;       // 空闲超时（秒）
    int  keep_alive_interval;   // 探测间隔（秒）
    int  keep_alive_count;      // 重试次数
} axk_transport_keep_alive_t;
```

---

## 函数接口

### `axk_transport_list_init`

创建传输列表（用于多协议管理）。

```c
axk_transport_list_handle_t axk_transport_list_init(void);
```

---

### `axk_transport_list_destroy`

销毁传输列表及所有子传输。

```c
axk_err_t axk_transport_list_destroy(axk_transport_list_handle_t list);
```

---

### `axk_transport_list_add`

添加传输到列表并绑定 scheme。

```c
axk_err_t axk_transport_list_add(axk_transport_list_handle_t list,
                                  axk_transport_handle_t t,
                                  const char *scheme);
```

| 参数 | 说明 |
|------|------|
| `list` | 传输列表句柄 |
| `t` | 传输句柄 |
| `scheme` | 协议标识（如 `"mqtt"` `"wss"`） |

---

### `axk_transport_list_get_transport`

通过 scheme 获取对应传输。

```c
axk_transport_handle_t axk_transport_list_get_transport(
    axk_transport_list_handle_t list,
    const char *scheme);
```

---

### `axk_transport_connect`

连接到服务器。

```c
int axk_transport_connect(axk_transport_handle_t t,
                           const char *host,
                           int port,
                           int timeout_ms);
```

| 参数 | 说明 |
|------|------|
| `host` | 服务器地址（域名或 IP） |
| `port` | 端口号 |
| `timeout_ms` | 连接超时（毫秒） |

**返回值**：0=成功，-1=失败

---

### `axk_transport_read`

读取数据。

```c
int axk_transport_read(axk_transport_handle_t t,
                        char *buffer,
                        int len,
                        int timeout_ms);
```

| 参数 | 说明 |
|------|------|
| `buffer` | 接收缓冲区 |
| `len` | 缓冲区大小 |
| `timeout_ms` | 超时 |

**返回值**：>0=读取字节数，-1=错误，0=对端关闭

---

### `axk_transport_write`

发送数据。

```c
int axk_transport_write(axk_transport_handle_t t,
                         const char *buffer,
                         int len,
                         int timeout_ms);
```

**返回值**：>=0=发送字节数，-1=错误

---

### `axk_transport_poll_read`

等待可读事件。

```c
int axk_transport_poll_read(axk_transport_handle_t t, int timeout_ms);
```

| 返回值 | 说明 |
|--------|------|
| 0 | 超时 |
| -1 | 错误 |
| >0 | 可读 |

---

### `axk_transport_poll_write`

等待可写事件。

```c
int axk_transport_poll_write(axk_transport_handle_t t, int timeout_ms);
```

---

### `axk_transport_close`

关闭传输连接。

```c
int axk_transport_close(axk_transport_handle_t t);
```

---

### `axk_transport_destroy`

销毁传输句柄。

```c
int axk_transport_destroy(axk_transport_handle_t t);
```

---

### `axk_transport_get_context_data`

获取用户上下文数据。

```c
void *axk_transport_get_context_data(axk_transport_handle_t t);
```

---

### `axk_transport_set_context_data`

设置用户上下文数据。

```c
axk_err_t axk_transport_set_context_data(axk_transport_handle_t t,
                                          void *data);
```

---

### `axk_transport_get_payload_transport_handle`

获取载荷传输句柄（如从 SSL 获取底层 TCP）。

```c
axk_transport_handle_t axk_transport_get_payload_transport_handle(
    axk_transport_handle_t t);
```

---

### `axk_transport_get_error_handle`

获取错误描述符。

```c
axk_tls_error_handle_t axk_transport_get_error_handle(axk_transport_handle_t t);
```

---

### `axk_transport_get_errno`

获取并清除最后 socket 错误码。

```c
int axk_transport_get_errno(axk_transport_handle_t t);
```

---

### `axk_transport_set_func`

设置传输层函数指针（高级用法）。

```c
axk_err_t axk_transport_set_func(axk_transport_handle_t t,
                                  connect_func _connect,
                                  io_read_func _read,
                                  io_func _write,
                                  trans_func _close,
                                  poll_func _poll_read,
                                  poll_func _poll_write,
                                  trans_func _destroy);
```

---

### `axk_transport_set_async_connect_func`

设置异步连接函数。

```c
axk_err_t axk_transport_set_async_connect_func(
    axk_transport_handle_t t,
    connect_async_func _connect_async_func);
```

---

## SSL 传输（axk_transport_ssl.h）

```c
#include "axk_transport_ssl.h"

// 创建 SSL 传输
axk_transport_handle_t axk_transport_ssl_init(void);

// 设置服务器 CA 证书（PEM）
void axk_transport_ssl_set_cert_data(axk_transport_handle_t t,
                                      const char *data, int len);

// 设置 DER 格式证书
void axk_transport_ssl_set_cert_data_der(axk_transport_handle_t t,
                                          const char *data, int len);

// 启用全局 CA 证书池
void axk_transport_ssl_enable_global_ca_store(axk_transport_handle_t t);

// 设置客户端证书（双向认证）
void axk_transport_ssl_set_client_cert_data(axk_transport_handle_t t,
                                             const char *data, int len);

// 设置客户端私钥
void axk_transport_ssl_set_client_key_data(axk_transport_handle_t t,
                                             const char *data, int len);

// 设置私钥密码
void axk_transport_ssl_set_client_key_password(axk_transport_handle_t t,
                                                const char *password,
                                                int password_len);

// 设置 ALPN 协议
void axk_transport_ssl_set_alpn_protocol(axk_transport_handle_t t,
                                           const char **alpn_protos);

// 跳过 CN 验证（不推荐）
void axk_transport_ssl_skip_common_name_check(axk_transport_handle_t t);

// 使用安全芯片（ATECC608A）
void axk_transport_ssl_use_secure_element(axk_transport_handle_t t);

// 设置 Keep-Alive
void axk_transport_ssl_set_keep_alive(axk_transport_handle_t t,
                                        axk_transport_keep_alive_t *cfg);
```

---

## 使用示例

### 多协议传输列表

```c
axk_transport_list_handle_t list = axk_transport_list_init();

// 添加 TCP 传输
axk_transport_handle_t tcp = axk_transport_tcp_init();
axk_transport_list_add(list, tcp, "tcp");

// 添加 SSL 传输
axk_transport_handle_t ssl = axk_transport_ssl_init();
axk_transport_ssl_set_cert_data(ssl, ca_cert, strlen(ca_cert) + 1);
axk_transport_list_add(list, ssl, "ssl");

// 按 scheme 获取并使用
axk_transport_handle_t transport = axk_transport_list_get_transport(list, "ssl");
axk_transport_connect(transport, "example.com", 443, 5000);
axk_transport_write(transport, "Hello", 5, 5000);
```

### 基本 TCP 连接

```c
axk_transport_handle_t tcp = axk_transport_tcp_init();
int ret = axk_transport_connect(tcp, "192.168.1.100", 8080, 5000);
if (ret == 0) {
    char buf[256];
    int len = axk_transport_read(tcp, buf, sizeof(buf), 3000);
    if (len > 0) {
        // 处理数据
    }
    axk_transport_close(tcp);
}
```
