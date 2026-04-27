# HTTP Server (HTTPD) API 参考

> 来源文件：`components/network/httpd/include/httpd.h`  
> BL602 内置轻量级 HTTP 服务器，支持 CGI 脚本和静态文件服务。

---

## 概述

BL602 HTTPD 工作模式：

```
浏览器/客户端  ──HTTP──▶  BL602 HTTPD  ──▶  CGI 脚本处理
                              │
                              └──▶  静态文件 (SPI Flash)
```

**特点**：
- 集成在 LwIP 协议栈之上（基于 `altcp`）
- 支持 CGI（Common Gateway Interface）回调
- 支持 URI 路由注册
- 支持 POST 数据处理

---

## 头文件

```c
#include "httpd.h"
```

---

## CGI 回调

### `tCGI`

CGI 处理函数类型定义。

```c
typedef err_t (*tCGI)(struct httpd_state *pVars, char *pBuffer, int iBufferLen);
```

| 参数 | 说明 |
|------|------|
| `pVars` | HTTP 请求状态结构（包含请求方法、URI、参数、POST 数据等） |
| `pBuffer` | 输出缓冲区，用于写入响应内容 |
| `iBufferLen` | 缓冲区大小 |

**返回值**：`ERR_OK`（成功）或其他 LwIP err_t

---

### `httpd_register_cgi`

注册 CGI 路由。

```c
void httpd_register_cgi(const char *pcMyURI, tCGI pFunction);
```

| 参数 | 说明 |
|------|------|
| `pcMyURI` | URI 路径，如 `"/api/led"` |
| `pFunction` | 处理该 URI 的 CGI 函数 |

---

## SSI 回调

### `tSSIHandler`

SSI（Server Side Include）处理函数类型。

```c
typedef u16_t (*tSSIHandler)(const char *pcInsert, char *pBuffer, u16_t iBufferLen);
```

---

### `httpd_register_ssi_handler`

注册 SSI 处理函数。

```c
void httpd_register_ssi_handler(tSSIHandler pHandler,
                                const char **ppcTags,
                                u16_t uiNumTags);
```

| 参数 | 说明 |
|------|------|
| `pHandler` | SSI 处理函数 |
| `ppcTags` | SSI 标签数组（如 `{"temp", "humidity"}`） |
| `uiNumTags` | 标签数量 |

---

## 初始化

### `httpd_init`

初始化并启动 HTTP 服务器。

```c
void httpd_init(void);
```

> 默认监听 `0.0.0.0:80`。调用此函数后，HTTPD 开始接收 HTTP 请求。

---

### `httpd_set_port`

设置监听端口（需在 `httpd_init` 前调用）。

```c
void httpd_set_port(uint16_t port);
```

---

## CGI 使用示例

### CGI 回调函数实现

```c
#include "httpd.h"

static err_t cgi_led_handler(struct httpd_state *pVars,
                              char *pBuffer, int iBufferLen)
{
    const char *method = pVars->request_method; // "GET" 或 "POST"

    if (strcmp(method, "GET") == 0) {
        // GET /api/led：返回当前 LED 状态
        snprintf(pBuffer, iBufferLen,
                 "HTTP/1.1 200 OK\r\n"
                 "Content-Type: application/json\r\n"
                 "Connection: close\r\n"
                 "\r\n"
                 "{\"led\":%d}",
                 led_state);
    } else if (strcmp(method, "POST") == 0) {
        // POST /api/led：解析 body 设置 LED 状态
        char *body = pVars->pcPostData; // POST 请求体
        // 解析 body，设置为 LED
        snprintf(pBuffer, iBufferLen,
                 "HTTP/1.1 200 OK\r\n"
                 "Content-Type: text/plain\r\n"
                 "\r\n"
                 "OK");
    }

    return ERR_OK;
}

static err_t cgi_sensor_handler(struct httpd_state *pVars,
                                 char *pBuffer, int iBufferLen)
{
    // 读取传感器数据
    int temp = read_temperature();
    int humi = read_humidity();

    snprintf(pBuffer, iBufferLen,
             "HTTP/1.1 200 OK\r\n"
             "Content-Type: application/json\r\n"
             "Cache-Control: no-cache\r\n"
             "\r\n"
             "{\"temp\":%d,\"humidity\":%d}",
             temp, humi);

    return ERR_OK;
}
```

### 注册 CGI 并启动

```c
void app_main(void)
{
    // 其他初始化...

    // 注册 CGI 路由
    httpd_register_cgi("/api/led", cgi_led_handler);
    httpd_register_cgi("/api/sensor", cgi_sensor_handler);

    // 启动 HTTP 服务器
    httpd_init();
    printf("HTTP server started on port 80\r\n");
}
```

## SSI 使用示例

```c
static u16_t ssi_handler(const char *pcInsert, char *pBuffer, u16_t iBufferLen)
{
    if (strcmp(pcInsert, "temp") == 0) {
        return snprintf(pBuffer, iBufferLen, "%d", read_temperature());
    } else if (strcmp(pcInsert, "humidity") == 0) {
        return snprintf(pBuffer, iBufferLen, "%d", read_humidity());
    }
    return 0;
}

static const char *tags[] = {"temp", "humidity"};

void app_main(void)
{
    httpd_register_ssi_handler(ssi_handler, tags, 2);
    httpd_init();
}
```

HTML 模板中使用 `<!--#temp-->` 会被替换为实际温度值。

## httpd_state 关键字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `request_method` | `char *` | 请求方法："GET" 或 "POST" |
| `uri` | `char *` | 请求 URI 路径 |
| `pcGetVars` | `char *` | URL 查询参数（GET 请求） |
| `pcPostData` | `char *` | POST 请求体数据 |
| `iPostDataLen` | `int` | POST 数据长度 |
| `remote_ip` | `u32_t` | 客户端 IP 地址 |
| `remote_port` | `u16_t` | 客户端端口 |
