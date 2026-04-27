# TLS/SSL API 参考

> 来源文件：`components/network/axk_protocol_stack/axk_tls/axk_tls.h`  
> 基于 mbedTLS 的 TLS 1.2/TLS 1.3 支持，提供证书验证、PSK 认证、会话恢复等功能。

---

## 概述

AXK-TLS 是 BL602 的传输层安全库，基于 mbedTLS，支持：
- TLS 客户端/服务器
- 证书认证 + PSK 认证
- 客户端双向认证
- TLS 会话恢复
- 全局 CA 证书池

---

## 头文件

```c
#include "axk_tls.h"
```

---

## 连接状态

```c
typedef enum axk_tls_conn_state {
    AXK_TLS_INIT = 0,       // 初始化
    AXK_TLS_CONNECTING,     // 连接中
    AXK_TLS_HANDSHAKE,      // 握手中
    AXK_TLS_FAIL,           // 失败
    AXK_TLS_DONE,           // 完成
} axk_tls_conn_state_t;
```

---

## TLS 配置结构

### `axk_tls_cfg_t` — 客户端配置

```c
typedef struct axk_tls_cfg {
    // ALPN 协议列表（用于 HTTP/2）
    const char **alpn_protos;

    // CA 证书（PEM 或 DER）
    union {
        const unsigned char *cacert_buf;
        const unsigned char *cacert_pem_buf;  // 兼容旧名称
    };
    union {
        unsigned int cacert_bytes;
        unsigned int cacert_pem_bytes;
    };

    // 客户端证书（双向认证）
    union {
        const unsigned char *clientcert_buf;
        const unsigned char *clientcert_pem_buf;
    };
    union {
        unsigned int clientcert_bytes;
        unsigned int clientcert_pem_bytes;
    };

    // 客户端私钥
    union {
        const unsigned char *clientkey_buf;
        const unsigned char *clientkey_pem_buf;
    };
    union {
        unsigned int clientkey_bytes;
        unsigned int clientkey_pem_bytes;
    };

    const unsigned char *clientkey_password;  // 私钥密码
    unsigned int clientkey_password_len;

    bool non_block;                          // 非阻塞模式
    bool use_secure_element;                 // 使用 ATECC608A 安全芯片
    int timeout_ms;                          // 超时（毫秒）
    bool use_global_ca_store;                // 使用全局 CA 证书池
    const char *common_name;                 // 服务器 CN 验证
    bool skip_common_name;                   // 跳过 CN 验证
    tls_keep_alive_cfg_t *keep_alive_cfg;   // TCP Keep-Alive
    const psk_hint_key_t *psk_hint_key;    // PSK 认证
    axk_err_t (*crt_bundle_attach)(void *conf); // 证书_bundle
    void *ds_data;                           // 数字签名参数
    bool is_plain_tcp;                       // 明文 TCP（不加密）
    struct ifreq *if_name;                   // 网络接口
} axk_tls_cfg_t;
```

### `psk_hint_key_t` — PSK 认证

```c
typedef struct psk_key_hint {
    const uint8_t *key;      // PSK 密钥（二进制）
    const size_t key_size;   // 密钥长度
    const char *hint;       // PSK 提示字符串
} psk_hint_key_t;
```

---

## 函数接口

### `axk_tls_init`

创建 TLS 上下文。

```c
axk_tls_t *axk_tls_init(void);
```

**返回值**：成功返回 TLS 句柄，失败返回 NULL

---

### `axk_tls_conn_new_sync`

同步方式创建 TLS 连接（阻塞）。

```c
int axk_tls_conn_new_sync(const char *hostname,
                           int hostlen,
                           int port,
                           const axk_tls_cfg_t *cfg,
                           axk_tls_t *tls);
```

| 参数 | 说明 |
|------|------|
| `hostname` | 服务器域名 |
| `hostlen` | 域名长度 |
| `port` | 端口号 |
| `cfg` | TLS 配置（可为 NULL 表示明文 TCP） |
| `tls` | TLS 句柄 |

**返回值**：1=成功，0=连接进行中，-1=失败

---

### `axk_tls_conn_http_new_sync`

通过 URL 创建 TLS 连接（同步）。

```c
int axk_tls_conn_http_new_sync(const char *url,
                                const axk_tls_cfg_t *cfg,
                                axk_tls_t *tls);
```

---

### `axk_tls_conn_new_async`

异步方式创建 TLS 连接（非阻塞）。

```c
int axk_tls_conn_new_async(const char *hostname,
                            int hostlen,
                            int port,
                            const axk_tls_cfg_t *cfg,
                            axk_tls_t *tls);
```

---

### `axk_tls_conn_http_new_async`

通过 URL 异步创建 TLS 连接。

```c
int axk_tls_conn_http_new_async(const char *url,
                                 const axk_tls_cfg_t *cfg,
                                 axk_tls_t *tls);
```

---

### `axk_tls_conn_write`

通过 TLS 连接发送数据。

```c
ssize_t axk_tls_conn_write(axk_tls_t *tls, const void *data, size_t datalen);
```

---

### `axk_tls_conn_read`

通过 TLS 连接接收数据。

```c
ssize_t axk_tls_conn_read(axk_tls_t *tls, void *data, size_t datalen);
```

---

### `axk_tls_conn_destroy`

关闭 TLS 连接并释放资源。

```c
int axk_tls_conn_destroy(axk_tls_t *tls);
```

**返回值**：0 成功，-1 失败

---

### `axk_tls_get_bytes_avail`

获取当前记录层中剩余可读字节数。

```c
ssize_t axk_tls_get_bytes_avail(axk_tls_t *tls);
```

---

### `axk_tls_get_conn_sockfd`

获取 TLS 连接的底层 socket 描述符。

```c
axk_err_t axk_tls_get_conn_sockfd(axk_tls_t *tls, int *sockfd);
```

---

### `axk_tls_plain_tcp_connect`

创建明文 TCP 连接（通过 TLS 层）。

```c
axk_err_t axk_tls_plain_tcp_connect(const char *host,
                                     int hostlen,
                                     int port,
                                     const axk_tls_cfg_t *cfg,
                                     axk_tls_error_handle_t error_handle,
                                     int *sockfd);
```

---

## 全局 CA 证书池

### `axk_tls_init_global_ca_store`

初始化全局 CA 证书池。

```c
axk_err_t axk_tls_init_global_ca_store(void);
```

---

### `axk_tls_set_global_ca_store`

设置全局 CA 证书（PEM 格式）。

```c
axk_err_t axk_tls_set_global_ca_store(const unsigned char *cacert_pem_buf,
                                       const unsigned int cacert_pem_bytes);
```

---

### `axk_tls_free_global_ca_store`

释放全局 CA 证书池。

```c
void axk_tls_free_global_ca_store(void);
```

---

### `axk_tls_get_global_ca_store`

获取当前全局 CA 证书池指针。

```c
mbedtls_x509_crt *axk_tls_get_global_ca_store(void);
```

---

## 错误处理

### `axk_tls_get_and_clear_last_error`

获取并清除最后的 TLS 错误。

```c
axk_err_t axk_tls_get_and_clear_last_error(axk_tls_error_handle_t h,
                                             int *axk_tls_code,
                                             int *axk_tls_flags);
```

---

### `axk_tls_get_and_clear_error_type`

获取指定类型的错误并清除。

```c
axk_err_t axk_tls_get_and_clear_error_type(axk_tls_error_handle_t h,
                                             axk_tls_error_type_t err_type,
                                             int *error_code);
```

---

## 使用示例

### 简单 HTTPS 请求

```c
#include "axk_tls.h"

static const char *request =
    "GET / HTTP/1.1\r\n"
    "Host: www.example.com\r\n"
    "User-Agent: BL602\r\n"
    "Connection: close\r\n"
    "\r\n";

void https_get_example(void)
{
    axk_tls_t *tls = axk_tls_init();
    if (!tls) return;

    axk_tls_cfg_t cfg = {
        .cacert_buf = (const unsigned char *)ca_cert_pem,
        .cacert_bytes = strlen(ca_cert_pem) + 1,
        .timeout_ms = 10000,
    };

    int ret = axk_tls_conn_new_sync("www.example.com", 16, 443, &cfg, tls);
    if (ret != 1) {
        printf("TLS connect failed: %d\r\n", ret);
        axk_tls_conn_destroy(tls);
        return;
    }

    // 发送请求
    axk_tls_conn_write(tls, request, strlen(request));

    // 读取响应
    char buf[1024];
    ssize_t len;
    while ((len = axk_tls_conn_read(tls, buf, sizeof(buf) - 1)) > 0) {
        buf[len] = '\0';
        printf("%s", buf);
    }

    axk_tls_conn_destroy(tls);
}
```

### PSK 认证模式

```c
static const uint8_t psk_key[] = { 0x01, 0x02, 0x03, 0x04 };
static const char psk_hint[] = "BL602_DEVICE";

psk_hint_key_t psk = {
    .key = psk_key,
    .key_size = sizeof(psk_key),
    .hint = psk_hint,
};

axk_tls_cfg_t cfg = {
    .psk_hint_key = &psk,
};
```

---

## TLS Keep-Alive 配置

```c
typedef struct tls_keep_alive_cfg {
    bool keep_alive_enable;      // 使能
    int keep_alive_idle;        // 空闲超时（秒）
    int keep_alive_interval;    // 探测间隔（秒）
    int keep_alive_count;        // 重试次数
} tls_keep_alive_cfg_t;
```
