# lwIP HTTPD Server API Reference (BL616/BL618)

## 概述

BL616/BL618 的 bouffalo_sdk 在 `components/net/lwip/` 中内置了 lwIP HTTPD Server，支持 **HTTP 服务器**、**CGI 动态路由**、**SSI 标签替换**、**POST 数据接收** 等完整功能。该实现基于瑞典计算机科学研究所的轻量级 HTTPd，经过 Texas Instruments 扩展了 SSI/CGI 能力。

**所在路径**：`components/net/lwip/lwip/src/include/lwip/apps/`

---

## 头文件

| 文件 | 说明 |
|------|------|
| `lwip/apps/httpd.h` | HTTPD 核心 API（CGI/SSI/HTTP Server） |
| `lwip/apps/httpd_opts.h` | 功能开关和配置宏 |
| `lwip/apps/fs.h` | 文件系统抽象层（fs_open/fs_read/fs_close） |
| `lwip/apps/fs.c` | 默认 RAM 文件系统实现 |

---

## 初始化

### httpd_init()

启动 HTTP 服务器（监听 80 端口）：

```c
#include "lwip/apps/httpd.h"

void httpd_init(void);
```

**示例**：

```c
void app_main(void)
{
    // 初始化 LwIP 协议栈（参考 lwip.md）
    tcpip_init(lwip_init_done, NULL);
}

static void lwip_init_done(void *arg)
{
    // LwIP 初始化完成后启动 HTTP 服务器
    httpd_init();
    printf("HTTP server started on port 80\r\n");
}
```

### httpd_inits() — HTTPS 服务器

启动带 TLS 加密的 HTTPS 服务器：

```c
#include "lwip/apps/httpd.h"
#include "lwip/apps/altcp_tls.h"

void httpd_inits(struct altcp_tls_config *conf);
```

**参数**：
- `conf` — mbedtls TLS 配置（通过 `altcp_tls_create_config_server()` 创建）

---

## CGI — Common Gateway Interface（动态 URL）

CGI 用于处理**动态 URL 路由**，例如 `/api/led/on`、`/api/sensor/read`。

### tCGIHandler — CGI 处理函数类型

```c
typedef const char *(*tCGIHandler)(int iIndex,
                                   int iNumParams,
                                   char *pcParam[],
                                   char *pcValue[]);
```

**参数**：
- `iIndex` — CGI 处理器索引（对应注册顺序）
- `iNumParams` — URI 参数个数
- `pcParam[]` — 参数名数组（如 `"state"`）
- `pcValue[]` — 参数值数组（如 `"on"`）

**返回值**：要返回的页面路径，如 `"/thanks.html"` 或 `"/response/error.ssi"`

**示例**：

```c
const char *handle_led_control(int iIndex, int iNumParams,
                                 char *pcParam[], char *pcValue[])
{
    for (int i = 0; i < iNumParams; i++) {
        if (strcmp(pcParam[i], "state") == 0) {
            if (strcmp(pcValue[i], "on") == 0) {
                bflb_gpio_set(gpio, GPIO_PIN_0);  // LED 亮
            } else {
                bflb_gpio_reset(gpio, GPIO_PIN_0);  // LED 灭
            }
        }
    }
    return "/led_response.html";
}
```

### tCGI — CGI 路由结构

```c
typedef struct {
    const char *pcCGIName;     // URL 路径，如 "/api/led"
    tCGIHandler pfnCGIHandler; // 处理函数
} tCGI;
```

### http_set_cgi_handlers() — 注册 CGI 路由

```c
void http_set_cgi_handlers(const tCGI *pCGIs, int iNumHandlers);
```

**示例**：

```c
const tCGI httpRouter[] = {
    { "/api/led",      handle_led_control },
    { "/api/sensor",   handle_sensor_read },
    { "/api/relay",    handle_relay },
};

void app_main(void)
{
    // ...
    httpd_init();
    http_set_cgi_handlers(httpRouter, sizeof(httpRouter) / sizeof(httpRouter[0]));
}
```

---

## SSI — Server Side Include（嵌入式标签）

SSI 用于在 **HTML 文件中嵌入动态变量**，标签形式为 `<!--#varname-->`。

### tSSIHandler — SSI 标签回调类型

```c
typedef u16_t (*tSSIHandler)(
#if LWIP_HTTPD_SSI_RAW
    const char *ssi_tag_name,  // 标签名（非数组索引）
#else
    int iIndex,               // 标签在数组中的索引
#endif
    char *pcInsert,           // 输出缓冲区（填入替换文本）
    int iInsertLen           // 输出缓冲区长度
    /* ... 可选参数 ... */
);
```

**返回值**：
- 写入 `pcInsert` 的字符数（不含 `\0`）
- `HTTPD_SSI_TAG_UNKNOWN (0xFFFF)` — 标签未识别

**示例**：

```c
static const char *ssi_tags[] = { "UPTIME", "TEMP", "LED_STATE" };

u16_t ssi_handler(int iIndex, char *pcInsert, int iInsertLen)
{
    switch (iIndex) {
        case 0: // UPTIME
            return snprintf(pcInsert, iInsertLen, "%lu seconds",
                            xTaskGetTickCount() / configTICK_RATE_HZ);
        case 1: // TEMP
            return snprintf(pcInsert, iInsertLen, "%.1f", read_temperature());
        case 2: // LED_STATE
            return snprintf(pcInsert, iInsertLen, "%s",
                            led_is_on() ? "ON" : "OFF");
        default:
            return HTTPD_SSI_TAG_UNKNOWN;
    }
}

void app_main(void)
{
    httpd_init();
    http_set_ssi_handler(ssi_handler, ssi_tags,
                         sizeof(ssi_tags) / sizeof(ssi_tags[0]));
}
```

**HTML 中使用**：

```html
<!-- index.shtml -->
<html>
<body>
  <p>Uptime: <!--#UPTIME--></p>
  <p>Temperature: <!--#TEMP--> °C</p>
  <p>LED: <!--#LED_STATE--></p>
  <form action="/api/led" method="get">
    <button name="state" value="on">LED ON</button>
    <button name="state" value="off">LED OFF</button>
  </form>
</body>
</html>
```

### http_set_ssi_handler() — 注册 SSI 处理器

```c
void http_set_ssi_handler(tSSIHandler pfnSSIHandler,
                          const char **ppcTags,
                          int iNumTags);
```

---

## POST 请求处理

### httpd_post_begin() — POST 请求开始

```c
err_t httpd_post_begin(void *connection,
                       const char *uri,
                       const char *http_request,
                       u16_t http_request_len,
                       int content_len,
                       char *response_uri,
                       u16_t response_uri_len,
                       u8_t *post_auto_wnd);
```

**返回值**：`ERR_OK` 接受请求，其他拒绝。

### httpd_post_receive_data() — 接收 POST 数据

```c
err_t httpd_post_receive_data(void *connection, struct pbuf *p);
```

**注意**：收到数据后**必须自行释放 pbuf**：

```c
err_t httpd_post_receive_data(void *connection, struct pbuf *p)
{
    // 处理 p->payload 中的数据
    pbuf_free(p);  // 必须释放
    return ERR_OK;
}
```

### httpd_post_finished() — POST 完成

```c
void httpd_post_finished(void *connection,
                         char *response_uri,
                         u16_t response_uri_len);
```

**说明**：在 `response_uri` 中填入响应页面路径，如 `"/upload_ok.html"`。

---

## 文件系统抽象层（fs.h）

HTTPD 通过文件系统抽象层提供静态文件，CGI/SSI 处理完成后也需要返回文件给客户端。

### fs_open() — 打开文件

```c
#include "lwip/apps/fs.h"

struct fs_file {
    const void *data;   // 文件数据指针
    size_t len;         // 文件长度
    int index;          // 当前读取位置
    void *state;       // 状态指针（用于自定义数据）
    // ...
};

err_t fs_open(struct fs_file *file, const char *name);
```

**返回**：`ERR_OK` 成功，其他失败。

### fs_read() — 读取文件

```c
int fs_read(struct fs_file *file, char *buffer, int count);
```

**返回**：实际读取的字节数，0 表示文件结束，-1 表示错误。

### fs_close() — 关闭文件

```c
void fs_close(struct fs_file *file);
```

### 使用 netconn 发送文件

SDK 常用模式（CGI 处理完后发送文件）：

```c
#include "lwip/apps/fs.h"
#include "lwip/netconn.h"

struct netconn *http_active_conn;  // 全局或通过 connection_state 传递

void send_file_to_client(const char *path)
{
    struct fs_file file;
    if (fs_open(&file, path) == ERR_OK) {
        netconn_write(http_active_conn,
                      file.data,
                      file.len,
                      NETCONN_NOCOPY);
        fs_close(&file);
    }
}
```

---

## 配置选项（lwipopts_user.h）

在 `lwipopts_user.h` 中启用 HTTPD 相关功能：

```c
// ==================== HTTPD 配置 ====================

// 启用 CGI（动态 URL）
#define LWIP_HTTPD_CGI           1

// 启用 SSI（服务器端标签）
#define LWIP_HTTPD_SSI           1

// 启用 POST 请求支持
#define LWIP_HTTPD_SUPPORT_POST   1

// POST 手动窗口管理（throttle 接收速度）
#define LWIP_HTTPD_POST_MANUAL_WND  1

// 最大 CGI 参数个数
#define LWIP_HTTPD_MAX_CGI_PARAMETERS  16

// SSI 最大标签数
#define LWIP_HTTPD_MAX_SSI_TAGS        8

// HTTP 服务器监听端口
#define LWIP_HTTPD_SERVER_PORT          80

// 启用 HTTPS
#define HTTPD_ENABLE_HTTPS              1

// 文件系统根目录（RAM 文件系统使用）
#define LWIP_HTTPD_FSDATA_TYPE          1

// 最大 URI 长度
#define LWIP_HTTPD_MAX_URI_LENGTH       256

// TCP 发送缓冲（影响大文件传输）
#define HTTP_MAX_OUTPUT_LEN             4096
```

**注意**：启用 CGI/SSI 后必须**注册对应的处理函数**，否则访问对应 URL 会返回 404。

---

## 完整示例：从 Flash 提供 HTML 页面

### 1. 准备 HTML 文件（放入 fsdata.c）

通常使用 Python 工具将 HTML/CSS/JS 打包为 `fsdata.c`（参考 `tools/makefsdata`）。简化方案：用 RAM 文件系统。

### 2. 代码实现

```c
#include "FreeRTOS.h"
#include "task.h"
#include "lwip/tcpip.h"
#include "lwip/netif.h"
#include "lwip/apps/httpd.h"
#include "lwip/apps/fs.h"
#include "wifi_mgmr.h"

static struct netif *sta_netif;

// ==================== CGI 处理器 ====================
const char *cgi_led_handler(int iIndex, int iNumParams,
                            char *pcParam[], char *pcValue[])
{
    for (int i = 0; i < iNumParams; i++) {
        if (strcmp(pcParam[i], "state") == 0) {
            if (strcmp(pcValue[i], "on") == 0) {
                bflb_gpio_set(gpio, GPIO_PIN_0);
            } else {
                bflb_gpio_reset(gpio, GPIO_PIN_0);
            }
        }
    }
    return "/led_ok.html";
}

const tCGI cgi_handlers[] = {
    { "/api/led", cgi_led_handler },
};

// ==================== SSI 处理器 ====================
static const char *ssi_tags[] = { "IP", "STATUS" };

u16_t ssi_handler(int iIndex, char *pcInsert, int iInsertLen)
{
    switch (iIndex) {
        case 0: // IP
            return snprintf(pcInsert, iInsertLen, "%s",
                            ipaddr_ntoa(&sta_netif->ip_addr));
        case 1: // STATUS
            return snprintf(pcInsert, iInsertLen, "Connected");
        default:
            return HTTPD_SSI_TAG_UNKNOWN;
    }
}

// ==================== 任务 ====================
static void web_server_task(void *param)
{
    vTaskDelay(pdMS_TO_TICKS(500));  // 等待网络就绪

    httpd_init();
    http_set_cgi_handlers(cgi_handlers,
                          sizeof(cgi_handlers) / sizeof(cgi_handlers[0]));
    http_set_ssi_handler(ssi_handler, ssi_tags,
                         sizeof(ssi_tags) / sizeof(ssi_tags[0]));

    printf("HTTP server running at http://%s/\r\n",
           ipaddr_ntoa(&sta_netif->ip_addr));

    vTaskDelete(NULL);
}

void app_main(void)
{
    // ... Wi-Fi 连接代码 ...
    // wifi_mgmr_sta_connect(...);

    // 创建 HTTP 服务器任务
    xTaskCreate(web_server_task, "httpd", 1024, NULL, 5, NULL);
}
```

### 3. HTML 页面示例（index.shtml）

```html
<!DOCTYPE html>
<html>
<head>
  <title>BL618 Web Server</title>
  <style>
    body { font-family: Arial; text-align: center; padding: 40px; }
    .status { font-size: 24px; margin: 20px 0; }
    button { padding: 10px 30px; font-size: 18px; margin: 5px; }
    .on { background: #4CAF50; color: white; }
    .off { background: #f44336; color: white; }
  </style>
</head>
<body>
  <h1>BL618 Web Server</h1>
  <p>IP: <!--#IP--></p>
  <p>Status: <!--#STATUS--></p>
  <hr>
  <h2>LED Control</h2>
  <form action="/api/led" method="get">
    <button class="on" name="state" value="on">LED ON</button>
    <button class="off" name="state" value="off">LED OFF</button>
  </form>
</body>
</html>
```

---

## SDK 示例路径

| 示例 | 路径 |
|------|------|
| EMAC HTTP Server | `examples/peripherals/emac/lwip_http_server/` |
| Wi-Fi RESTful API | `examples/wifi/sta/http_restful_api/` |

---

## 注意事项

1. **必须先 `httpd_init()` 再注册 CGI/SSI**：否则路由不会生效
2. **RAM 文件系统限制**：默认 `fsdata.c` 中的文件编译进固件，文件太大会占满 Flash
3. **Wi-Fi 模式下**：需确保 `netif` 已 up 且获取到 IP 后再启动 HTTP 服务器
4. **lwIP 线程安全**：HTTPD 在 TCPIP 线程中运行，注册 CGI/SSI 需在 `httpd_init()` 前完成
5. **POST 需要手动 `pbuf_free`**：收到数据后必须释放，否则内存泄漏
