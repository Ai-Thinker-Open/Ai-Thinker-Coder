# HTTP Client (HTTPC) API 参考

> 来源文件：`components/network/httpd/include/http_client.h`  
> 基于 LwIP altcp 的 HTTP 客户端实现，支持 HTTPS（TLS）。

---

## 概述

BL602 HTTPC 支持：
- HTTP/HTTPS GET、POST 请求
- 自定义请求头
- 请求体发送（POST）
- 响应头和响应体读取
- 连接超时配置

---

## 头文件

```c
#include "http_client.h"
```

---

## 类型定义

### `httpc_request_header`

HTTP 请求头。

```c
typedef struct {
    const char *name;    // 请求头名称（如 "Content-Type"）
    const char *value;   // 请求头值
} httpc_request_header_t;
```

### `httpc_response`

HTTP 响应结构。

```c
typedef struct {
    int status_code;          // HTTP 状态码（如 200、404）
    uint32_t content_length;  // Content-Length（若有）
    char *header_data;        // 响应头数据（需外部释放）
} httpc_response_t;
```

### `httpc_cb`

响应数据回调函数类型。

```c
typedef err_t (*httpc_cb)(void *arg,
                            struct altcp_pcb *pcb,
                            struct pbuf *p,
                            err_t err);
```

---

## 函数接口

### `httpc_get`

发送 HTTP GET 请求。

```c
err_t httpc_get(struct altcp_pcb *pcb,
                const char *url,
                const httpc_request_header_t *headers,
                int num_headers,
                void *callback_arg,
                httpc_cb resp_fn,
                httpc_response_t *resp);
```

---

### `httpc_post`

发送 HTTP POST 请求。

```c
err_t httpc_post(struct altcp_pcb *pcb,
                 const char *url,
                 const char *content_type,
                 const void *body,
                 size_t body_len,
                 const httpc_request_header_t *headers,
                 int num_headers,
                 void *callback_arg,
                 httpc_cb resp_fn,
                 httpc_response_t *resp);
```

---

### `httpc_request`

通用 HTTP 请求（支持自定义方法）。

```c
err_t httpc_request(struct altcp_pcb *pcb,
                    const char *url,
                    const char *method,
                    const char *content_type,
                    const void *body,
                    size_t body_len,
                    const httpc_request_header_t *headers,
                    int num_headers,
                    void *callback_arg,
                    httpc_cb resp_fn,
                    httpc_response_t *resp);
```

---

## 使用示例

### 简单 GET 请求

```c
#include "http_client.h"

static err_t my_resp_cb(void *arg, struct altcp_pcb *pcb,
                        struct pbuf *p, err_t err)
{
    if (p != NULL) {
        // 打印响应内容
        char *data = (char *)p->payload;
        printf("Response (%d bytes): %.*s\r\n", p->len, p->len, data);
        altcp_recved(pcb, p->len); // 通知协议栈已处理数据
        pbuf_free(p);
    } else {
        // 响应结束
        printf("Response complete\r\n");
    }
    return ERR_OK;
}

void http_get_example(void)
{
    struct altcp_pcb *pcb = altcp_new(NULL); // 使用默认 PCB
    if (!pcb) return;

    httpc_response_t resp;
    err_t err = httpc_get(pcb,
                           "http://httpbin.org/get",
                           NULL, 0,        // 无额外请求头
                           NULL,           // callback_arg
                           my_resp_cb,     // 响应回调
                           &resp);         // 响应结构

    if (err != ERR_OK) {
        printf("HTTP request failed: %d\r\n", err);
    }
}
```

### POST 请求（JSON 数据）

```c
static err_t post_resp_cb(void *arg, struct altcp_pcb *pcb,
                          struct pbuf *p, err_t err)
{
    if (p != NULL) {
        printf("POST response: %.*s\r\n", p->len, (char *)p->payload);
        altcp_recved(pcb, p->len);
        pbuf_free(p);
    } else {
        printf("POST complete\r\n");
    }
    return ERR_OK;
}

void http_post_example(void)
{
    struct altcp_pcb *pcb = altcp_new(NULL);

    const char *json_body = "{\"name\":\"BL602\",\"value\":123}";
    httpc_request_header_t headers[] = {
        {"Content-Type", "application/json"},
        {"Authorization", "Bearer token123"},
    };

    httpc_response_t resp;
    err_t err = httpc_post(pcb,
                            "http://httpbin.org/post",
                            "application/json",
                            json_body,
                            strlen(json_body),
                            headers,
                            2,
                            NULL,
                            post_resp_cb,
                            &resp);

    if (err == ERR_OK) {
        printf("HTTP status: %d\r\n", resp.status_code);
    }
}
```
