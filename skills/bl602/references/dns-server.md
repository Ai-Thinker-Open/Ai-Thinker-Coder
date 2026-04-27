# DNS Server API 参考

> 来源文件：`components/network/dns_server/include/dns_server.h`  
> BL602 本地 DNS 服务器，支持将特定域名解析到本地 IP（用于 Captive Portal、域名重定向等场景）。

---

## 概述

DNS Server 用于在 BL602 开启 AP 模式时，劫持所有 DNS 查询并返回指定的 IP 地址。典型应用：
- Captive Portal（强制门户）
- 域名重定向
- 物联网本地控制（域名固定为设备地址）

---

## 头文件

```c
#include "dns_server.h"
```

---

## 函数接口

### `dns_server_init`

启动 DNS 服务器。

```c
void *dns_server_init(void);
```

**返回值**：DNS 服务器句柄（失败返回 NULL）

---

### `dns_server_deinit`

停止并销毁 DNS 服务器。

```c
void dns_server_deinit(void *server);
```

| 参数 | 说明 |
|------|------|
| `server` | `dns_server_init` 返回的句柄 |

---

## 使用示例

### Captive Portal 场景

```c
#include "dns_server.h"

static void *g_dns_server = NULL;

void captive_portal_start(const char *redirect_ip)
{
    // 启动 DNS 服务器（将所有域名解析到 redirect_ip）
    g_dns_server = dns_server_init();
    if (g_dns_server == NULL) {
        printf("DNS server init failed\r\n");
        return;
    }
    printf("DNS server started, redirecting to %s\r\n", redirect_ip);
}

void captive_portal_stop(void)
{
    if (g_dns_server) {
        dns_server_deinit(g_dns_server);
        g_dns_server = NULL;
        printf("DNS server stopped\r\n");
    }
}

// 在 app_main 中启动
void app_main(void)
{
    // 开启 AP 模式...
    // 启动 DNS 重定向（将所有域名解析到 AP 的 IP）
    captive_portal_start("192.168.4.1");
}
```

### 本地物联网控制

```c
void iot_control_start(void)
{
    // 设备作为 AP，手机连上后访问 mydevice.local 域名
    // DNS Server 将 mydevice.local 解析到 192.168.4.1
    g_dns_server = dns_server_init();
}
```

---

## 注意事项

- DNS Server 只能在 AP 模式下使用
- 所有非本地网络的 DNS 查询都会被劫持
- 如需精确控制哪些域名被劫持，需修改 SDK 源码
