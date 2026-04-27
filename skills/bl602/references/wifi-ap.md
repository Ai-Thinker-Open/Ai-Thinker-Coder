# Wi-Fi API 参考（AP 模式）

> 来源文件：`components/network/wifi_manager/bl60x_wifi_driver/include/wifi_mgmr_ext.h`

---

## 使能与启动

### `wifi_mgmr_ap_enable`

使能 AP 模式，获取接口句柄。

```c
wifi_interface_t wifi_mgmr_ap_enable(void);
```

**返回值**：Wi-Fi 接口句柄

---

### `wifi_mgmr_ap_stop`

停止 AP。

```c
int wifi_mgmr_ap_stop(wifi_interface_t *interface);
```

---

### `wifi_mgmr_ap_start`

启动 AP 热点。

```c
int wifi_mgmr_ap_start(wifi_interface_t *interface,
                        char *ssid,
                        int hidden_ssid,
                        char *passwd,
                        int channel);
```

| 参数 | 说明 |
|------|------|
| `interface` | `wifi_mgmr_ap_enable` 返回的句柄 |
| `ssid` | 热点名称 |
| `hidden_ssid` | `1` = 隐藏 SSID，`0` = 广播 SSID |
| `passwd` | 密码（NULL 表示开放网络） |
| `channel` | 信道（1~13） |

---

### `wifi_mgmr_ap_start_adv`

启动 AP（高级版，支持 DHCP 配置）。

```c
int wifi_mgmr_ap_start_adv(wifi_interface_t *interface,
                            char *ssid,
                            int hidden_ssid,
                            char *passwd,
                            int channel,
                            uint8_t use_dhcp);
```

---

### `wifi_mgmr_ap_start_atcmd`

通过 AT 命令方式启动 AP。

```c
int wifi_mgmr_ap_start_atcmd(wifi_interface_t *interface,
                              char *ssid,
                              int hidden_ssid,
                              char *passwd,
                              int channel,
                              int max_sta_supported);
```

---

## AP IP 配置

### `wifi_mgmr_ap_ip_set`

设置 AP IP 地址。

```c
int wifi_mgmr_ap_ip_set(uint32_t ip, uint32_t gw, uint32_t mask);
```

---

### `wifi_mgmr_ap_ip_get`

获取 AP IP 地址。

```c
int wifi_mgmr_ap_ip_get(uint32_t *ip, uint32_t *gw, uint32_t *mask);
```

---

### `wifi_mgmr_ap_mac_get`

获取 AP MAC 地址。

```c
int wifi_mgmr_ap_mac_get(uint8_t mac[6]);
```

---

## DHCP 服务器

### `wifi_mgmr_ap_dhcp_enable`

使能 AP 内置 DHCP 服务器。

```c
int wifi_mgmr_ap_dhcp_enable(void);
```

---

### `wifi_mgmr_ap_dhcp_disable`

禁用 AP 内置 DHCP 服务器。

```c
int wifi_mgmr_ap_dhcp_disable(void);
```

---

### `wifi_mgmr_ap_dhcp_range_set`

配置 DHCP 地址池范围。

```c
int wifi_mgmr_ap_dhcp_range_set(uint32_t ip, uint32_t mask, int start, int end);
```

---

### `wifi_mgmr_ap_dhcp_range_get`

获取 DHCP 地址池配置。

```c
int wifi_mgmr_ap_dhcp_range_get(uint32_t *ip, uint32_t *mask, int *start, int *end);
```

---

## 已连接终端管理

### `wifi_mgmr_ap_sta_cnt_get`

获取已连接的终端数量。

```c
int wifi_mgmr_ap_sta_cnt_get(uint8_t *sta_cnt);
```

---

### `wifi_mgmr_ap_sta_info_get`

获取指定终端的信息。

```c
int wifi_mgmr_ap_sta_info_get(struct wifi_sta_basic_info *sta_info, uint8_t idx);
```

---

### `wifi_mgmr_ap_sta_delete`

踢出指定终端。

```c
int wifi_mgmr_ap_sta_delete(uint8_t sta_idx);
```

---

### `wifi_mgmr_ap_set_gateway`

设置 AP 上行网关。

```c
int wifi_mgmr_ap_set_gateway(char *gateway);
```

---

## 信道切换

### `wifi_mgmr_ap_chan_switch`

切换 AP 信道。

```c
int wifi_mgmr_ap_chan_switch(wifi_interface_t *interface, int channel, uint8_t cs_count);
```

---

## 其他配置

### `wifi_mgmr_conf_max_sta`

设置最大连接终端数。

```c
int wifi_mgmr_conf_max_sta(uint8_t max_sta_supported);
```

---

### `wifi_mgmr_beacon_interval_set`

设置 Beacon 间隔。

```c
int wifi_mgmr_beacon_interval_set(uint16_t beacon_int);
```

---

## 使用示例

```c
#include "wifi_mgmr_ext.h"

// 使能 AP 模式
wifi_interface_t ap_if = wifi_mgmr_ap_enable();

// 设置 AP IP（必须先设置 IP 再启动 DHCP）
wifi_mgmr_ap_ip_set(0xC0A80101, 0xC0A80101, 0xFFFFFF00); // 192.168.1.1

// 启用 DHCP
wifi_mgmr_ap_dhcp_enable();
wifi_mgmr_ap_dhcp_range_set(0xC0A80101, 0xFFFFFF00, 100, 200);

// 启动热点：SSID="BL602_AP", 信道 6, 密码 "12345678"
int ret = wifi_mgmr_ap_start(ap_if, "BL602_AP", 0, "12345678", 6);
if (ret == 0) {
    printf("AP started on channel 6\r\n");
}

// 查看已连接终端
uint8_t sta_cnt = 0;
wifi_mgmr_ap_sta_cnt_get(&sta_cnt);
printf("Connected stations: %d\r\n", sta_cnt);

// 获取终端信息
struct wifi_sta_basic_info sta_info;
for (int i = 0; i < sta_cnt; i++) {
    wifi_mgmr_ap_sta_info_get(&sta_info, i);
    printf("STA[%d] MAC: %02X:%02X:... RSSI: %d\r\n",
           i, sta_info.sta_mac[0], sta_info.sta_mac[1], sta_info.rssi);
}

// 停止 AP
wifi_mgmr_ap_stop(&ap_if);
```
