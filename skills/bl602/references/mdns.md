# mDNS 响应器 API 参考

> 来源文件：`components/network/lwip_mdns/mdns_server.h`  
> 多播 DNS（mDNS）响应器，支持本地网络服务发现。

---

## 概述

mDNS 用于在无 DNS 服务器的局域网中发现服务和设备。通过 mDNS，设备可通过 `hostname.local` 域名互相发现，无需配置中心服务器。

---

## 头文件

```c
#include "lwip/apps/mdns_opts.h"
#include "lwip/netif.h"
```

> 需在 `lwipopts.h` 中开启 `LWIP_MDNS_RESPONDER=1`。

---

## 类型定义

### `mdns_name_result_cb_t`

名称探测结果回调：

```c
typedef void (*mdns_name_result_cb_t)(struct netif *netif, u8_t result);
// result: MDNS_PROBING_CONFLICT(0) 或 MDNS_PROBING_SUCCESSFUL(1)
```

### `service_get_txt_fn_t`

服务 TXT 记录回调：

```c
typedef void (*service_get_txt_fn_t)(struct mdns_service *service, void *txt_userdata);
```

---

## 函数接口

### `mdns_resp_init`

初始化 mDNS 响应器。

```c
void mdns_resp_init(void);
```

---

### `mdns_resp_deinit`

反初始化 mDNS。

```c
void mdns_resp_deinit(void);
```

---

### `mdns_resp_register_name_result_cb`

注册名称冲突检测回调。

```c
void mdns_resp_register_name_result_cb(mdns_name_result_cb_t cb);
```

---

### `mdns_resp_add_netif`

为网络接口注册主机名。

```c
err_t mdns_resp_add_netif(struct netif *netif, const char *hostname, u32_t dns_ttl);
```

| 参数 | 说明 |
|------|------|
| `netif` | 网络接口 |
| `hostname` | 主机名（如 `"my-device"`） |
| `dns_ttl` | DNS 记录 TTL（秒） |

---

### `mdns_resp_remove_netif`

从接口移除 mDNS。

```c
err_t mdns_resp_remove_netif(struct netif *netif);
```

---

### `mdns_resp_rename_netif`

修改接口的主机名。

```c
err_t mdns_resp_rename_netif(struct netif *netif, const char *hostname);
```

---

### `mdns_resp_add_service`

注册 mDNS 服务。

```c
s8_t mdns_resp_add_service(struct netif *netif,
                           const char *name,
                           const char *service,
                           enum mdns_sd_proto proto,
                           u16_t port,
                           u32_t dns_ttl,
                           service_get_txt_fn_t txt_fn,
                           void *txt_userdata);
```

| 参数 | 说明 |
|------|------|
| `netif` | 网络接口 |
| `name` | 服务实例名（如 `"My HTTP Server"`） |
| `service` | 服务类型（如 `"_http"`） |
| `proto` | 协议：`MDNS_SD_PROTO_TCP` 或 `MDNS_SD_PROTO_UDP` |
| `port` | 服务端口 |
| `dns_ttl` | TTL |
| `txt_fn` | TXT 记录回调（可 NULL） |
| `txt_userdata` | 传递给回调的用户数据 |

**返回值**：>=0=服务槽号（用于删除/修改），<0=失败

---

### `mdns_resp_del_service`

删除已注册服务。

```c
err_t mdns_resp_del_service(struct netif *netif, s8_t slot);
```

---

### `mdns_resp_rename_service`

修改服务实例名。

```c
err_t mdns_resp_rename_service(struct netif *netif, s8_t slot, const char *name);
```

---

### `mdns_resp_add_service_txtitem`

向服务添加 TXT 记录项。

```c
err_t mdns_resp_add_service_txtitem(struct mdns_service *service,
                                    const char *txt, u8_t txt_len);
```

---

### `mdns_responder_start`

启动 mDNS 响应器。

```c
int mdns_responder_start(struct netif *netif);
```

---

### `mdns_responder_stop`

停止 mDNS 响应器。

```c
int mdns_responder_stop(struct netif *netif);
```

---

### `mdns_resp_restart`

重启 mDNS 响应（触发重新探测）。

```c
void mdns_resp_restart(struct netif *netif);
```

---

### `mdns_resp_announce`

主动广播（通知网络设置变化）。

```c
void mdns_resp_announce(struct netif *netif);
```

---

## 使用示例

### 基本 HTTP 服务注册

```c
#include "lwip/apps/mdns_opts.h"
#include "lwip/netif.h"

static void http_txt_callback(struct mdns_service *service, void *txt_userdata)
{
    (void)service; (void)txt_userdata;
    mdns_resp_add_service_txtitem(service, "path=/", 7);
    mdns_resp_add_service_txtitem(service, "version=1.0", 12);
}

void mdns_http_example(struct netif *netif)
{
    mdns_resp_init();

    // 注册设备名
    mdns_resp_add_netif(netif, "my-wb2", 120);

    // 注册 HTTP 服务
    mdns_resp_add_service(netif,
                          "My HTTP Server",
                          "_http",
                          MDNS_SD_PROTO_TCP,
                          80,
                          120,
                          http_txt_callback,
                          NULL);

    mdns_responder_start(netif);
}
```

### 在 WiFi 连接后启动

```c
void wifi_connected_callback(struct netif *netif)
{
    if (!netif_is_up(netif)) return;

    mdns_resp_add_netif(netif, "ai-wb2", 120);
    mdns_resp_add_service(netif, "WB2 Device", "_device-info",
                          MDNS_SD_PROTO_TCP, 8080, 120, NULL, NULL);
    mdns_responder_start(netif);
}
```
