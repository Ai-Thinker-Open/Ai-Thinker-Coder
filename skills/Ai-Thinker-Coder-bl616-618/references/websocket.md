# librws WebSocket 客户端库

## 概述

librws 是一个轻量级 WebSocket 客户端库，专为资源受限的嵌入式环境设计，支持单线程异步 I/O 模式。该库适用于 IoT 设备与云端服务器之间的双向通信场景，具有体积小、接口简洁、依赖少等特点。库内部集成了 mbedTLS 支持，可直接建立 wss（WebSocket Secure）加密连接。

当前版本：**1.2.4**

## 基础类型

### rws_bool 布尔类型

librws 定义了自定义布尔类型 `rws_bool`，作为无符号字符类型（unsigned char）的别名。

```c
typedef unsigned char rws_bool;
#define rws_true  1
#define rws_false 0
```

所有返回布尔状态的函数均使用此类型，判断时直接使用 `if (result == rws_true)` 或简写为 `if (result)`。

### rws_handle 句柄类型

库中所有对象句柄均定义为 `void *` 类型，包括：

- `rws_socket` - Socket 句柄
- `rws_error` - 错误对象句柄
- `rws_mutex` - 互斥锁句柄
- `rws_thread` - 线程句柄

## Socket 连接管理

### 创建与连接

**rws_socket_create()**

创建新的 WebSocket Socket 对象，返回句柄供后续操作使用。创建成功后需设置 URL 参数并调用 `rws_socket_connect()` 发起连接。

```c
rws_socket rws_socket_create(void);
```

**rws_socket_set_url() / rws_socket_connect()**

设置连接目标并发起连接。URL 由 scheme、host、port、path 四部分组成。

```c
void rws_socket_set_url(rws_socket socket,
                         const char * scheme,    // "ws" 或 "wss"
                         const char * host,       // 服务器域名或 IP
                         const int port,          // 端口号
                         const char * path);      // 路径，如 "/"

rws_bool rws_socket_connect(rws_socket socket);
```

也可分步设置各参数：

```c
rws_socket_set_scheme(socket, "wss");      // 设置协议 scheme
rws_socket_set_host(socket, "example.com"); // 设置主机
rws_socket_set_port(socket, 8443);        // 设置端口
rws_socket_set_path(socket, "/ws");       // 设置路径
```

**连接状态查询**

```c
rws_bool rws_socket_is_connected(rws_socket socket);
// 返回 rws_true 表示已建立连接并完成握手
```

### 断开与释放

**rws_socket_disconnect_and_release() / rws_socket_delete()**

关闭 Socket 连接并释放资源。调用后该 Socket 句柄不可再使用。

```c
void rws_socket_disconnect_and_release(rws_socket socket);
void rws_socket_delete(rws_socket socket);
```

## 消息发送

librws 提供多种发送接口，支持文本和二进制数据。

### 文本消息发送

**rws_socket_send_text() / rws_socket_send_text2()**

发送文本帧到已连接的服务器。`send_text2` 允许指定长度。

```c
rws_bool rws_socket_send_text(rws_socket socket, const char * text);
rws_bool rws_socket_send_text2(rws_socket socket, const char * text, size_t len);
```

### 二进制消息发送

二进制数据发送采用分帧接口，分为开始、继续、结束三个阶段，适用于大数据量传输场景：

```c
// 开始发送二进制数据（发送帧头）
rws_bool rws_socket_send_bin_start(rws_socket socket, const char * bin, size_t len);

// 继续发送二进制数据
rws_bool rws_socket_send_bin_continue(rws_socket socket, const char * bin, size_t len);

// 结束发送（发送帧尾）
rws_bool rws_socket_send_bin_finish(rws_socket socket, const char * bin, size_t len);
```

### 发送相关常量

**RWS_MAX_SEND_APPEND_SIZE**

定义了单次发送操作的最大缓冲大小：

```c
#define RWS_MAX_SEND_APPEND_SIZE (1024 * 640)  // 640KB
```

内部实现中，`send_append_size` 默认等于此值，标识发送队列的缓冲区上限。

### 低层发送接口

内部 `rws_socket_send()` 函数提供原始数据发送能力：

```c
rws_bool rws_socket_send(rws_socket socket, const void * data, const size_t data_size);
```

## 消息接收

librws 采用回调模式处理接收数据，需预先注册回调函数。

### 接收回调注册

**rws_socket_on_received_text() / rws_socket_on_received_bin()**

```c
typedef void (*rws_on_socket_recvd_text)(rws_socket socket,
                                         const char * text,
                                         const unsigned int length,
                                         bool is_finished);

typedef void (*rws_on_socket_recvd_bin)(rws_socket socket,
                                         const void * data,
                                         const unsigned int length,
                                         bool is_finished);

void rws_socket_set_on_received_text(rws_socket socket, rws_on_socket_recvd_text callback);
void rws_socket_set_on_received_bin(rws_socket socket, rws_on_socket_recvd_bin callback);
```

回调参数中的 `is_finished` 标识当前帧是否为消息的最后一帧（用于分片传输场景）。

## 连接状态回调

### 连接建立回调

**rws_socket_on_connected()**

当 WebSocket 握手成功完成后触发。

```c
typedef void (*rws_on_socket)(rws_socket socket);

void rws_socket_set_on_connected(rws_socket socket, rws_on_socket callback);
```

### 断开连接回调

**rws_socket_on_disconnected()**

当连接关闭时触发，包括主动断开和异常断开发。

```c
void rws_socket_set_on_disconnected(rws_socket socket, rws_on_socket callback);
```

### Pong 回调

收到服务器 Pong 响应时触发（用于心跳检测）：

```c
typedef void (*rws_on_socket_recvd_pong)(rws_socket socket);
void rws_socket_set_on_received_pong(rws_socket socket, rws_on_socket_recvd_pong callback);
```

## 线程管理

librws 内部使用工作线程处理网络 I/O，库同时提供线程创建接口供应用层使用。

### rws_thread_create()

创建并立即启动新线程：

```c
typedef void (*rws_thread_funct)(void * user_object);

RWS_API(rws_thread) rws_thread_create(rws_thread_funct thread_function, void * user_object);
```

### rws_thread_join()

等待线程结束并获取返回值：

```c
RWS_API(int) rws_thread_join(rws_thread thread, void ** retval);
```

### rws_thread_sleep()

线程睡眠（毫秒级）：

```c
RWS_API(void) rws_thread_sleep(const unsigned int millisec);
```

## 错误处理

### 获取错误

```c
rws_error rws_socket_get_error(rws_socket socket);
int rws_error_get_code(rws_error error);
const char * rws_error_get_description(rws_error error);
int rws_error_get_http_error(rws_error error);
```

### 错误码定义

| 错误码 | 含义 |
|--------|------|
| rws_error_code_none | 无错误 |
| rws_error_code_missed_parameter | 缺少必要参数 |
| rws_error_code_send_handshake | 发送握手失败 |
| rws_error_code_parse_handshake | 解析握手响应失败 |
| rws_error_code_read_write_socket | Socket 读写错误 |
| rws_error_code_connect_to_host | 连接主机失败 |
| rws_error_code_connection_closed | 连接被关闭 |

## SSL/TLS 支持

启用 SSL 时（`WEBSOCKET_SSL_ENABLE`），可设置服务器证书：

```c
void rws_socket_set_server_cert(rws_socket socket,
                                const char * server_cert,
                                int server_cert_len);
```

## 超时设置

```c
// 设置读取超时（毫秒）
int rws_socket_set_read_timeout(rws_socket socket, int timeout_ms);

// 设置写入超时（毫秒）
int rws_socket_set_write_timeout(rws_socket socket, int timeout_ms);
```

## 用户对象绑定

Socket 支持绑定用户自定义对象，便于在回调中识别上下文：

```c
void rws_socket_set_user_object(rws_socket socket, void * user_object);
void * rws_socket_get_user_object(rws_socket socket);
```

## WebSocket 与 MQTT 协议对比

| 特性 | WebSocket (librws) | MQTT |
|------|-------------------|------|
| 协议层级 | 应用层（HTTP 升级） | 应用层（独立协议） |
| 连接方式 | 持久化 TCP 长连接 | 持久化 TCP 长连接 |
| 通信模式 | 双向消息流 | 发布/订阅（Pub/Sub） |
| 消息模型 | 点对点，客户端-服务器 | 多对多，主题订阅 |
| 头部开销 | 较小（2-14 字节/帧） | 很小（2字节最小） |
| 适用场景 | 实时聊天、推送、页游 | IoT 传感器数据采集 |
| QoS 支持 | 需应用层实现 | 原生支持 QoS 0/1/2 |
| Broker 需求 | 普通 WebSocket 服务器 | MQTT Broker |
| 复杂度 | 简单 | 中等 |

**选择建议**：

- 简单请求-响应场景或需要透过防火墙的 Web 应用 → WebSocket
- 大规模设备数据采集、多对多消息分发 → MQTT
- 需要兼容现有 Web 技术栈 → WebSocket

## 代码示例

以下示例演示如何连接 wss:// 服务器并进行消息收发：

```c
#include "librws.h"
#include <stdio.h>

// 全局 socket 句柄
static rws_socket g_socket = NULL;

// 连接建立回调
static void on_connected(rws_socket socket) {
    printf("[WS] Connected\r\n");
    
    // 连接成功后发送文本消息
    const char *msg = "Hello, Server!";
    rws_socket_send_text(socket, msg);
}

// 接收文本回调
static void on_received_text(rws_socket socket, const char * text, 
                             const unsigned int length, bool is_finished) {
    printf("[WS] Received (%u bytes): %.*s\r\n", length, length, text);
}

// 断开连接回调
static void on_disconnected(rws_socket socket) {
    printf("[WS] Disconnected\r\n");
    g_socket = NULL;
}

// 初始化 WebSocket 客户端
int ws_client_init(void) {
    // 创建 socket 对象
    g_socket = rws_socket_create();
    if (!g_socket) {
        printf("[WS] Create failed\r\n");
        return -1;
    }
    
    // 设置回调
    rws_socket_set_on_connected(g_socket, on_connected);
    rws_socket_set_on_disconnected(g_socket, on_disconnected);
    rws_socket_set_on_received_text(g_socket, on_received_text);
    
    // 设置连接参数 (wss://echo.websocket.org:443/)
    rws_socket_set_url(g_socket, "wss", "echo.websocket.org", 443, "/");
    
    // 设置超时（可选）
    rws_socket_set_read_timeout(g_socket, 5000);
    rws_socket_set_write_timeout(g_socket, 5000);
    
    // 发起连接
    if (rws_socket_connect(g_socket) != rws_true) {
        printf("[WS] Connect failed\r\n");
        rws_socket_delete(g_socket);
        g_socket = NULL;
        return -1;
    }
    
    printf("[WS] Connecting...\r\n");
    return 0;
}

// 断开并清理
void ws_client_cleanup(void) {
    if (g_socket) {
        rws_socket_disconnect_and_release(g_socket);
        g_socket = NULL;
    }
}
```

### 二进制数据发送示例

```c
// 发送二进制数据（分帧）
static void send_binary_data(rws_socket socket, const uint8_t *data, size_t len) {
    size_t chunk_size = 1024;
    size_t offset = 0;
    
    while (offset < len) {
        size_t remain = len - offset;
        size_t send_len = (remain > chunk_size) ? chunk_size : remain;
        
        if (offset == 0) {
            // 开始帧
            rws_socket_send_bin_start(socket, (const char *)data, send_len);
        } else if (offset + send_len < len) {
            // 继续帧
            rws_socket_send_bin_continue(socket, (const char *)(data + offset), send_len);
        } else {
            // 结束帧
            rws_socket_send_bin_finish(socket, (const char *)(data + offset), send_len);
        }
        
        offset += send_len;
    }
}
```

## 参考

- [librws 官方仓库](https://github.com/nekopub/librws) - 轻量级 WebSocket 客户端库
- [WebSocket 协议 RFC 6455](https://tools.ietf.org/html/rfc6455) - WebSocket 标准协议规范
- Bouffalo SDK xwebsocket 组件源码：
  - `components/multimedia/xwebsocket/include/librws.h`
  - `components/multimedia/xwebsocket/include/rws_socket.h`
  - `components/multimedia/xwebsocket/include/rws_frame.h`
  - `components/multimedia/xwebsocket/include/rws_network.h`
