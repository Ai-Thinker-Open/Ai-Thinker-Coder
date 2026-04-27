# Wi-Fi API 参考（STA 模式）

> 来源文件：`components/network/wifi_manager/bl60x_wifi_driver/include/wifi_mgmr_ext.h`  
> 另一文件：`components/network/wifi_manager/bl60x_wifi_driver/wifi_mgmr_api.h`

> Wi-Fi 模块支持 Station（STA）和 Access Point（AP）两种模式。STA 模式下模块作为客户端连接路由器，AP 模式下模块作为热点供其他设备连接。

---

## 初始化与使能

### `wifi_mgmr_drv_init`

驱动初始化（系统启动时调用一次）。

```c
int wifi_mgmr_drv_init(wifi_conf_t *conf);
```

| 参数 | 说明 |
|------|------|
| `conf` | Wi-Fi 国家码和信道配置，见 `wifi_conf_t` |

**返回值**：`0` 成功

---

### `wifi_mgmr_init`

Wi-Fi 管理器初始化。

```c
int wifi_mgmr_init(void);
```

---

### `wifi_mgmr_sta_enable`

使能 STA 模式，获取 Wi-Fi 接口句柄。

```c
wifi_interface_t wifi_mgmr_sta_enable(void);
```

**返回值**：Wi-Fi 接口句柄（`wifi_interface_t`），用于后续连接操作

---

### `wifi_mgmr_sta_disable`

关闭 STA 模式。

```c
int wifi_mgmr_sta_disable(wifi_interface_t *interface);
```

---

## 连接操作

### `wifi_mgmr_sta_connect`

连接 Wi-Fi 热点（基础版本）。

```c
int wifi_mgmr_sta_connect(wifi_interface_t *wifi_interface,
                           char *ssid,
                           char *psk,
                           char *pmk,
                           uint8_t *mac,
                           uint8_t band,
                           uint8_t chan_id);
```

| 参数 | 说明 |
|------|------|
| `wifi_interface` | `wifi_mgmr_sta_enable` 返回的句柄 |
| `ssid` | 热点名称 |
| `psk` | 密码（可为 NULL 表示开放网络） |
| `pmk` | PSK 密钥（可为 NULL，自动计算） |
| `mac` | 目标 AP 的 MAC 地址（可为 NULL） |
| `band` | 频段（0 = 2.4G） |
| `chan_id` | 信道号（0 = 自动） |

**返回值**：`0` 成功

---

### `wifi_mgmr_sta_connect_ext`

连接热点（扩展版本，支持高级参数）。

```c
int wifi_mgmr_sta_connect_ext(wifi_interface_t *wifi_interface,
                                char *ssid,
                                char *passphr,
                                const ap_connect_adv_t *conn_adv_param);
```

---

### `wifi_mgmr_sta_disconnect`

断开 Wi-Fi 连接。

```c
int wifi_mgmr_sta_disconnect(void);
```

---

## 自动重连

### `wifi_mgmr_sta_autoconnect_enable`

使能自动重连。

```c
int wifi_mgmr_sta_autoconnect_enable(void);
```

---

### `wifi_mgmr_sta_autoconnect_disable`

禁用自动重连。

```c
int wifi_mgmr_sta_autoconnect_disable(void);
```

---

### `wifi_mgmr_sta_autoconnect_set`

配置自动重连间隔和次数。

```c
int wifi_mgmr_sta_autoconnect_set(int interval_second, int repeat_count);
```

---

## IP 地址

### `wifi_mgmr_sta_ip_set`

设置 STA IP 地址（手动模式）。

```c
int wifi_mgmr_sta_ip_set(uint32_t ip, uint32_t mask, uint32_t gw,
                          uint32_t dns1, uint32_t dns2);
```

---

### `wifi_mgmr_sta_ip_get`

获取 STA 当前 IP 地址。

```c
int wifi_mgmr_sta_ip_get(uint32_t *ip, uint32_t *gw, uint32_t *mask);
```

---

### `wifi_mgmr_sta_ip_unset`

清除 STA IP（恢复 DHCP）。

```c
int wifi_mgmr_sta_ip_unset(void);
```

---

### `wifi_mgmr_sta_netif_get`

获取 STA netif 接口（用于 LwIP 栈）。

```c
struct netif *wifi_mgmr_sta_netif_get(void);
```

---

## 状态与信息

### `wifi_mgmr_state_get`

获取 Wi-Fi 连接状态。

```c
int wifi_mgmr_state_get(int *state);
```

> 状态枚举：`WIFI_STATE_IDLE`、`WIFI_STATE_CONNECTING`、`WIFI_STATE_CONNECTED_IP_GOT`、`WIFI_STATE_DISCONNECT` 等

---

### `wifi_mgmr_rssi_get`

获取当前信号强度。

```c
int wifi_mgmr_rssi_get(int *rssi);
```

> 返回值：负数 dBm 值，如 `-65` 表示 -65dBm

---

### `wifi_mgmr_channel_get`

获取当前信道。

```c
int wifi_mgmr_channel_get(int *channel);
```

---

### `wifi_mgmr_sta_mac_get`

获取 STA MAC 地址。

```c
int wifi_mgmr_sta_mac_get(uint8_t mac[6]);
```

---

### `wifi_mgmr_set_country_code`

设置国家码（影响可用信道）。

```c
int wifi_mgmr_set_country_code(char *country_code);
```

> 示例：`wifi_mgmr_set_country_code("CN")`

---

### `wifi_mgmr_get_country_code`

获取当前国家码。

```c
int wifi_mgmr_get_country_code(char *country_code);
```

---

## 扫描

### `wifi_mgmr_scan`

扫描可用热点。

```c
int wifi_mgmr_scan(void *data, scan_complete_cb_t cb);
```

| 参数 | 说明 |
|------|------|
| `data` | 用户私有数据，传递给回调 |
| `cb` | 扫描完成回调 `scan_complete_cb_t` |

---

### `wifi_mgmr_scan_adv`

高级扫描（指定信道、SSID 等）。

```c
int wifi_mgmr_scan_adv(void *data,
                        scan_complete_cb_t cb,
                        uint16_t *channels,
                        uint16_t channel_num,
                        const uint8_t bssid[6],
                        const char *ssid,
                        uint8_t scan_mode,
                        uint32_t duration_scan);
```

---

### `wifi_mgmr_all_ap_scan`

获取所有扫描结果。

```c
int wifi_mgmr_all_ap_scan(wifi_mgmr_ap_item_t **ap_ary, uint32_t *num);
```

---

### `wifi_mgmr_cli_scanlist`

打印扫描列表到日志。

```c
int wifi_mgmr_cli_scanlist(void);
```

---

## 功耗管理

### `wifi_mgmr_sta_ps_enter`

进入低功耗模式。

```c
int wifi_mgmr_sta_ps_enter(uint32_t ps_level);
```

> `ps_level`：`PS_MODE_OFF`（关闭）、`PS_MODE_ON`（普通）、`PS_MODE_ON_DYN`（动态）

---

### `wifi_mgmr_sta_ps_exit`

退出低功耗模式。

```c
int wifi_mgmr_sta_ps_exit(void);
```

---

### `wifi_mgmr_set_wifi_active_time`

设置活跃时间。

```c
int wifi_mgmr_set_wifi_active_time(uint32_t ms);
```

---

### `wifi_mgmr_set_listen_interval`

设置监听间隔。

```c
int wifi_mgmr_set_listen_interval(uint16_t itv);
```

---

## 使用示例

```c
#include "wifi_mgmr_ext.h"

// Wi-Fi 初始化
wifi_conf_t conf = {
    .country_code = "CN",
    .channel_nums = 0,
};
wifi_mgmr_drv_init(&conf);
wifi_mgmr_init();

// 使能 STA 模式
wifi_interface_t wifi_if = wifi_mgmr_sta_enable();

// 连接热点（密码为空表示开放网络）
int ret = wifi_mgmr_sta_connect(wifi_if, "MySSID", "password", NULL, NULL, 0, 0);
if (ret == 0) {
    printf("Connecting...\r\n");
}

// 获取连接状态
int state;
wifi_mgmr_state_get(&state);
printf("Wi-Fi state: %d\r\n", state);

// 获取信号强度
int rssi;
wifi_mgmr_rssi_get(&rssi);
printf("RSSI: %d dBm\r\n", rssi);

// 断开连接
wifi_mgmr_sta_disconnect();

// 关闭 STA
wifi_mgmr_sta_disable(&wifi_if);
```
