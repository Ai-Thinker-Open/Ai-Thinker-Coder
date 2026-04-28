# HTTPS Client API Reference

## Overview

BL616/BL618 的 HTTPS 客户端基于 mbedTLS 实现，支持 TLS 1.2 和 TLS 1.3 协议，兼容 HTTP/1.1 keep-alive 长连接。该模块封装了底层的 TLS 握手、数据加密传输和 HTTP 协议解析，为嵌入式设备提供安全可靠的 HTTPS 通信能力。

HTTPS 客户端架构分为两层：

- **HTTP 层**：基于 Zephyr net/http 库实现 HTTP/1.1 协议解析和请求构建
- **TLS 层**：通过 https_wrapper 封装 mbedTLS，提供 SSL/TLS 加密通道

## Header Files

```c
#include "http/client.h"      // HTTP 客户端核心 API
#include "http/parser.h"      // HTTP 消息解析器
#include "https_client.h"     // HTTPS 客户端封装
#include "https_wrapper.h"    // mbedTLS 封装接口
```

## 常量与枚举

### HTTP_CRLF

```c
#define HTTP_CRLF "\r\n"
```

HTTP 协议行结束符，用于分隔 HTTP 头部的各个字段。

### HTTP_STATUS_STR_SIZE

```c
#define HTTP_STATUS_STR_SIZE 32
```

HTTP 状态码字符串数组大小，足够存储如 "200 OK"、"404 Not Found" 等标准状态描述。

### enum http_final_call

响应数据回调标志，指示当前数据是否是消息的最后一部分：

```c
enum http_final_call {
    HTTP_DATA_MORE = 0,   // 还有更多数据待接收
    HTTP_DATA_FINAL = 1   // 这是消息的最后数据
};
```

### enum http_method

支持的 HTTP 方法：

```c
enum http_method {
    HTTP_DELETE = 0,
    HTTP_GET = 1,
    HTTP_HEAD = 2,
    HTTP_POST = 3,
    HTTP_PUT = 4,
    HTTP_CONNECT = 5,
    HTTP_OPTIONS = 6,
    HTTP_TRACE = 7,
    HTTP_PATCH = 28
};
```

## 核心数据结构

### struct http_request

HTTP 请求结构体，描述一次 HTTP/HTTPS 请求的全部参数：

```c
struct http_request {
    /* 内部数据，应用层不应直接操作 */
    struct http_client_internal_data internal;

    /* === 以下字段需由用户填充 === */

    /** HTTP 方法：GET、POST、PUT 等 */
    enum http_method method;

    /** 响应回调函数，收到服务器响应时被调用（必填） */
    http_response_cb_t response;

    /** HTTP 解析器回调设置，用于获取解析过程中的详细信息（可选） */
    const struct http_parser_settings *http_cb;

    /** 用户提供的接收缓冲区，用于存储响应数据（必填） */
    uint8_t *recv_buf;

    /** 接收缓冲区长度（必填） */
    size_t recv_buf_len;

    /** 请求 URL 路径，例如 "/index.html" 或 "/api/data" */
    const char *url;

    /** 协议版本，通常为 "HTTP/1.1" */
    const char *protocol;

    /** NULL 结尾的 HTTP 头部字段列表 */
    const char **header_fields;

    /** Content-Type 头部值，如 "application/json" */
    const char *content_type_value;

    /** 目标主机名，如 "api.example.com"（必填） */
    const char *host;

    /** 端口号字符串，如 "443" 或 "8080"（可选） */
    const char *port;

    /** 载荷发送回调，当需要发送大量数据时使用（可选） */
    http_payload_cb_t payload_cb;

    /** 请求载荷数据，如 POST 请求的 body（可选） */
    const char *payload;

    /** 载荷长度，chunked 传输时设为 0 */
    size_t payload_len;

    /** 可选头部发送回调（可选） */
    http_header_cb_t optional_headers_cb;

    /** NULL 结尾的可选头部列表（可选） */
    const char **optional_headers;

    /** 取消回调，可用于中止正在进行的请求（可选） */
    http_cancel_cb_t cancel;
};
```

### struct http_response

HTTP 响应结构体，通过回调函数传递给应用层：

```c
struct http_response {
    /** HTTP 解析器设置 */
    const struct http_parser_settings *http_cb;

    /** 用户响应回调函数 */
    http_response_cb_t cb;

    /** 响应体数据起始地址（相对于 recv_buf） */
    uint8_t *body_frag_start;

    /** 响应体片段长度 */
    size_t body_frag_len;

    /** 接收缓冲区指针 */
    uint8_t *recv_buf;

    /** 接收缓冲区最大长度 */
    size_t recv_buf_len;

    /** 已接收数据的总长度（可能大于 recv_buf_len 表示被截断） */
    size_t data_len;

    /** Content-Length 头部值 */
    size_t content_length;

    /** 已处理（已交付给回调）的字节数 */
    size_t processed;

    /** HTTP 状态描述字符串，如 "200 OK" */
    char http_status[HTTP_STATUS_STR_SIZE];

    /** HTTP 状态码，3 位数字，如 200、404、500 */
    uint16_t http_status_code;

    /** Content-Range 信息 */
    struct http_content_range content_range;

    /** 标志位 */
    uint8_t cl_present : 1;        // Content-Length 头部存在
    uint8_t body_found : 1;        // 找到消息体
    uint8_t message_complete : 1;   // HTTP 消息解析完成
    uint8_t cr_present : 1;        // Content-Range 头部存在
};
```

### struct https_client_request

HTTPS 专用请求结构体，扩展了 TLS 证书配置：

```c
struct https_client_request {
    /* HTTP 相关字段（与 http_request 相同） */
    enum http_method method;
    http_response_cb_t response;
    const struct http_parser_settings *http_cb;
    const char *url;
    const char *protocol;
    const char **header_fields;
    const char *content_type_value;
    http_payload_cb_t payload_cb;
    const char *payload;
    size_t payload_len;
    http_header_cb_t optional_headers_cb;
    const char **optional_headers;
    http_cancel_cb_t cancel;

    /* === TLS 证书配置 === */

    /** 服务器 CA 证书（PEM 格式字符串），用于验证服务器身份 */
    const char *ca_pem;
    size_t ca_len;

    /** 客户端证书（PEM 格式），双向认证时使用 */
    const char *client_cert_pem;
    size_t client_cert_len;

    /** 客户端私钥（PEM 格式），双向认证时使用 */
    const char *client_key_pem;
    size_t client_key_len;

    /** HTTP 缓冲区大小（发送和接收共用）*/
    size_t buffer_size;
};
```

### struct http_wrapper_ssl_param_t

TLS 底层配置参数：

```c
typedef struct {
    const char *ca_cert;       // CA 证书
    int ca_cert_len;           // CA 证书长度
    const char *own_cert;      // 自身证书
    int own_cert_len;          // 自身证书长度
    const char *private_cert;  // 私钥
    int private_cert_len;      // 私钥长度

    const char **alpn;         // ALPN 协议列表
    int alpn_num;              // ALPN 协议数量

    const char *psk;           // 预共享密钥
    int psk_len;
    const char *pskhint;       // PSK 身份提示
    int pskhint_len;

    char *sni;                 // Server Name Indication
} http_wrapper_ssl_param_t;
```

## 回调函数类型

### http_response_cb_t

响应数据回调，HTTP 请求发送后服务器每返回一段数据都会触发：

```c
typedef void (*http_response_cb_t)(struct http_response *rsp,
                                   enum http_final_call final_data,
                                   void *user_data);
```

**参数说明：**
- `rsp`：包含响应数据的结构体指针
- `final_data`：`HTTP_DATA_MORE` 表示还有更多数据，`HTTP_DATA_FINAL` 表示这是最后一批数据
- `user_data`：调用者传入的私有数据

### http_payload_cb_t

载荷发送回调，用于分块发送大量数据：

```c
typedef int (*http_payload_cb_t)(int sock,
                                 struct http_request *req,
                                 void *user_data);
```

**返回值：**
- `>= 0`：已发送的字节数，继续发送
- `< 0`：错误码，终止请求

### http_header_cb_t

可选头部发送回调：

```c
typedef int (*http_header_cb_t)(int sock,
                                struct http_request *req,
                                void *user_data);
```

### http_cancel_cb_t

取消回调，可在中途终止请求：

```c
typedef int (*http_cancel_cb_t)(void *user_data);
```

**返回值：**
- 非零值：停止接收，返回 `-ECANCELED`
- 零：继续正常接收

## 核心 API

### http_client_req()

发送 HTTP 请求（异步回调模式）：

```c
int http_client_req(int sock, struct http_request *req,
                    int32_t timeout, void *user_data);
```

**参数：**
- `sock`：已建立的 socket 套接字
- `req`：HTTP 请求结构体指针
- `timeout`：超时时间（毫秒），`SYS_FOREVER_MS` 表示永不超时
- `user_data`：传递给回调的私有数据

**返回值：**
- `< 0`：错误码
- `>= 0`：已发送到服务器的字节数

**说明：**
调用者需先建立网络连接（TCP 或 TLS），该函数负责发送 HTTP 请求头和载荷，然后循环接收服务器响应并通过回调分发数据。

### https_client_request()

高级 HTTPS 请求接口，内部自动处理 TLS 连接：

```c
int https_client_request(const struct https_client_request *request,
                         uint32_t timeout, void *user_data);
```

**参数：**
- `request`：HTTPS 请求配置，包含 URL、方法、TLS 证书等
- `timeout`：超时时间（毫秒）
- `user_data`：私有数据，传递给回调

**返回值：**
- `< 0`：错误码
- `>= 0`：成功

**说明：**
此函数封装了 TLS 连接建立、证书验证、HTTP 请求发送和响应接收的全过程，是最常用的 HTTPS 接口。

### https_wrapper_connect()

建立 HTTPS 连接：

```c
https_wrapper_handle_t https_wrapper_connect(const char *host_name,
                                             uint16_t port,
                                             http_wrapper_ssl_param_t *param);
```

**参数：**
- `host_name`：目标主机名
- `port`：端口号（通常 443）
- `param`：TLS 参数，可为 NULL（HTTP 模式）

**返回值：**
- 非 NULL：有效的 HTTPS 连接句柄
- NULL：连接失败

### https_wrapper_destroy()

关闭 HTTPS 连接并释放资源：

```c
int https_wrapper_destroy(https_wrapper_handle_t https);
```

### https_wrapper_send()

通过 TLS 发送数据：

```c
int https_wrapper_send(https_wrapper_handle_t https,
                      const void *data, uint16_t size, int flags);
```

### https_wrapper_recv()

接收 TLS 数据：

```c
int https_wrapper_recv(https_wrapper_handle_t https,
                      uint8_t *data, uint32_t size, int flags);
```

### https_wrapper_socketfd_get()

获取底层 socket 文件描述符：

```c
int https_wrapper_socketfd_get(https_wrapper_handle_t https);
```

## mbedTLS SSL 上下文集成

HTTPS 客户端通过 https_wrapper 间接使用 mbedTLS。底层 mbedTLS 配置通过 `http_wrapper_ssl_param_t` 结构体传递，支持以下安全特性：

### TLS 版本支持

- TLS 1.2
- TLS 1.3

### 证书验证

**单向认证（仅验证服务器）：**
```c
ssl_param.ca_cert = server_ca_pem;
ssl_param.ca_cert_len = server_ca_len;
ssl_param.sni = hostname;  // 启用 SNI 主机名验证
```

**双向认证（同时验证客户端）：**
```c
ssl_param.own_cert = client_cert_pem;
ssl_param.own_cert_len = client_cert_len;
ssl_param.private_cert = client_key_pem;
ssl_param.private_cert_len = client_key_len;
```

### PSK 预共享密钥模式

适用于资源受限场景，省去证书开销：
```c
ssl_param.psk = psk_data;
ssl_param.psk_len = psk_len;
ssl_param.pskhint = psk_identity_hint;
```

### ALPN 协议协商

```c
const char *alpn_protos[] = { "http/1.1", "h2" };
ssl_param.alpn = alpn_protos;
ssl_param.alpn_num = 2;
```

## 代码示例

### 示例一：基础 HTTPS GET 请求

```c
#include "https_client.h"
#include <stdio.h>
#include <string.h>

#define RECV_BUFFER_SIZE 1024

static uint8_t recv_buf[RECV_BUFFER_SIZE];

/* 响应回调函数 */
static void response_callback(struct http_response *rsp,
                              enum http_final_call final_data,
                              void *user_data)
{
    if (rsp->body_frag_len > 0) {
        printf("[HTTP] Received %zu bytes, status: %s\r\n",
               rsp->body_frag_len, rsp->http_status);

        /* 处理响应体数据 */
        printf("Body: %.*s\r\n",
               (int)rsp->body_frag_len,
               rsp->body_frag_start);
    }

    if (final_data == HTTP_DATA_FINAL) {
        printf("[HTTP] Transfer complete, total: %zu bytes\r\n",
               rsp->processed);
    }
}

/* HTTPS GET 请求 */
int https_get_example(void)
{
    int ret;
    struct https_client_request request;

    memset(&request, 0, sizeof(request));

    request.method = HTTP_GET;
    request.url = "https://api.example.com/data";
    request.protocol = "HTTP/1.1";
    request.response = response_callback;

    /* 服务器证书验证（可选） */
    request.ca_pem = trusted_ca_pem;
    request.ca_len = trusted_ca_len;

    ret = https_client_request(&request, 30000, NULL);
    if (ret < 0) {
        printf("[HTTPS] Request failed: %d\r\n", ret);
        return ret;
    }

    return 0;
}
```

### 示例二：带自定义头部的 HTTPS POST 请求

```c
#include "https_client.h"
#include <stdio.h>
#include <string.h>

static uint8_t recv_buf[2048];
static const char *post_headers[] = {
    "X-Api-Key: your-api-key-here",
    "X-Request-Id: 12345",
    NULL
};

/* 响应回调 */
static void post_response_callback(struct http_response *rsp,
                                   enum http_final_call final_data,
                                   void *user_data)
{
    if (rsp->http_status_code >= 200 && rsp->http_status_code < 300) {
        printf("[HTTPS] Success: %s\r\n", rsp->http_status);
    } else {
        printf("[HTTPS] Error: %s (%d)\r\n",
               rsp->http_status, rsp->http_status_code);
    }

    if (rsp->body_found && rsp->body_frag_len > 0) {
        printf("[Response] %.*s\r\n",
               (int)rsp->body_frag_len,
               rsp->body_frag_start);
    }
}

/* HTTPS POST 请求示例 */
int https_post_example(void)
{
    int ret;
    struct https_client_request request;
    const char *json_payload = "{\"name\":\"test\",\"value\":123}";
    const char *content_type = "application/json";

    memset(&request, 0, sizeof(request));

    request.method = HTTP_POST;
    request.url = "https://api.example.com/endpoint";
    request.protocol = "HTTP/1.1";
    request.response = post_response_callback;
    request.recv_buf = recv_buf;
    request.recv_buf_len = sizeof(recv_buf);
    request.content_type_value = content_type;
    request.header_fields = post_headers;
    request.payload = json_payload;
    request.payload_len = strlen(json_payload);

    /* 启用服务器证书验证 */
    request.ca_pem = trusted_ca_pem;
    request.ca_len = trusted_ca_len;

    ret = https_client_request(&request, 30000, NULL);
    return ret;
}
```

### 示例三：分块接收大文件

```c
#include "https_client.h"
#include <stdio.h>
#include <fcntl.h>

#define CHUNK_SIZE 4096
static uint8_t recv_buf[CHUNK_SIZE];

/* 跟踪下载进度 */
static size_t total_downloaded = 0;

/* 响应回调 - 流式处理大文件 */
static void download_callback(struct http_response *rsp,
                              enum http_final_call final_data,
                              void *user_data)
{
    if (rsp->body_frag_len > 0) {
        /* 处理每个数据块 */
        total_downloaded += rsp->body_frag_len;

        printf("[Download] Progress: %zu / %zu bytes (%.1f%%)\r\n",
               total_downloaded,
               rsp->content_length,
               (double)total_downloaded / rsp->content_length * 100.0);
    }

    if (final_data == HTTP_DATA_FINAL) {
        printf("[Download] Complete: %zu bytes received\r\n",
               total_downloaded);
        total_downloaded = 0;
    }
}

/* 下载文件 */
int download_file(const char *url, const char *output_path)
{
    int ret;
    struct https_client_request request;

    memset(&request, 0, sizeof(request));

    request.method = HTTP_GET;
    request.url = url;
    request.protocol = "HTTP/1.1";
    request.response = download_callback;
    request.recv_buf = recv_buf;
    request.recv_buf_len = sizeof(recv_buf);
    request.ca_pem = trusted_ca_pem;
    request.ca_len = trusted_ca_len;

    ret = https_client_request(&request, 120000, NULL);
    return ret;
}
```

### 示例四：使用低层 API 自定义 TLS 配置

```c
#include "https_wrapper.h"
#include "http/client.h"
#include <stdio.h>

/* 自定义 TLS 参数 */
static http_wrapper_ssl_param_t ssl_config = {
    .ca_cert = trusted_ca_pem,
    .ca_cert_len = trusted_ca_len,
    .sni = "api.example.com",
    /* PSK 模式（可选）*/
    .psk = NULL,
    .psk_len = 0,
    /* ALPN */
    .alpn = (const char*[]){"http/1.1"},
    .alpn_num = 1,
};

/* 响应回调 */
static void response_cb(struct http_response *rsp,
                       enum http_final_call final_data,
                       void *user_data)
{
    if (rsp->body_frag_len > 0) {
        printf("%.*s", (int)rsp->body_frag_len, rsp->body_frag_start);
    }
    if (final_data == HTTP_DATA_FINAL) {
        printf("\r\n[Done]\r\n");
    }
}

int custom_tls_request(void)
{
    int sock;
    struct http_request req;
    uint8_t recv_buf[1024];

    /* 建立 HTTPS 连接 */
    https_wrapper_handle_t https = https_wrapper_connect(
        "api.example.com", 443, &ssl_config);
    if (!https) {
        printf("Connection failed\r\n");
        return -1;
    }

    sock = https_wrapper_socketfd_get(https);

    /* 构造 HTTP 请求 */
    memset(&req, 0, sizeof(req));
    req.method = HTTP_GET;
    req.url = "/api/data";
    req.protocol = "HTTP/1.1";
    req.host = "api.example.com";
    req.response = response_cb;
    req.recv_buf = recv_buf;
    req.recv_buf_len = sizeof(recv_buf);

    /* 发送请求 */
    int ret = http_client_req(sock, &req, 30000, NULL);

    /* 关闭连接 */
    https_wrapper_destroy(https);

    return ret;
}
```

## 错误处理

常见错误码：

| 错误码 | 含义 |
|--------|------|
| `-EINVAL` | 参数无效（空指针、缓冲区为 0 等） |
| `-ECONNREFUSED` | 连接被拒绝 |
| `-ECONNRESET` | 连接被对端重置 |
| `-ECANCELED` | 请求被取消回调终止 |
| `-ENOMEM` | 内存分配失败 |
| `-ETIMEDOUT` | 操作超时 |

## 注意事项

1. **Keep-alive**：HTTP/1.1 默认启用 keep-alive，同一连接可发送多个请求
2. **缓冲区大小**：根据预期响应大小合理设置 `recv_buf`，大响应建议分块处理
3. **TLS 证书**：生产环境务必验证服务器证书，测试环境可跳过
4. **超时设置**：网络不稳环境建议设置合理的超时时间
5. **线程安全**：回调函数可能在中断或专有线程上下文中调用，需确保线程安全

---

## 参考

- Bouffalo SDK `components/net/lib/http/` 源码
- `http/client.h` - HTTP 客户端核心 API 定义
- `https_client.h` - HTTPS 客户端封装接口
- `https_wrapper.h` - mbedTLS 封装接口
- `http/parser.h` - HTTP 消息解析器（基于 http_parser v2.7.1）
- [Zephyr net/http library](https://docs.zephyrproject.org/latest/connectivity/networking/api/http_client.html)
- [mbedTLS Documentation](https://mbedtls.readthedocs.io/)
