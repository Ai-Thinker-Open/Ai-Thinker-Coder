# BL616/BL618 Wi-Fi 6 (802.11ax) 技术文档

## 概述

BL616 和 BL618 是博流智能（Bouffalo）推出的高性能 Wi-Fi 6 无线芯片，完整支持 IEEE 802.11ax 标准（即 Wi-Fi 6）。与前代 802.11ac（Wi-Fi 5）相比，Wi-Fi 6 在吞吐量、延迟、功耗和密集网络环境性能等方面均有显著提升。

BL616/BL618 Wi-Fi 6 解决方案的核心特性包括：

| 特性 | 描述 |
|------|------|
| **MU-MIMO** | 多用户多输入多输出，支持上行/下行多用户数据传输 |
| **OFDMA** | 正交频分多址，将信道划分为更小的资源单元，提升频谱效率 |
| **BSS Color** | 基础服务集着色，有效减少同频干扰 |
| **TWT** | 目标唤醒时间，显著降低设备功耗 |
| **1024-QAM** | 更高阶调制，峰值速率提升约 25% |
| **8 空间流** | 支持最多 8 个空间流并发传输 |

## Wi-Fi 6 关键特性详解

### 调制与速率

Wi-Fi 6 采用 1024-QAM（1024 正交幅度调制），每个符号携带 10 bit 数据，相比 Wi-Fi 5 的 256-QAM（8 bit/符号）提升 25%。在 80 MHz 信道带宽下，理论峰值速率可达 1.2 Gbps；在 160 MHz 信道带宽下可达 2.4 Gbps。

### MU-MIMO 与 OFDMA

**MU-MIMO（多用户多输入多输出）** 允许 AP 同时与多个终端通信。在 BL616/BL618 实现中，支持下行 MU-MIMO 和上行 MU-MIMO。Wi-Fi 6 将 MU-MIMO 扩展至 8x8 天线配置，支持 8 空间流。

**OFDMA（正交频分多址）** 将信道划分为多个资源单元（RU），每个 RU 可分配给不同用户。OFDMA 显著提升了多用户并发场景下的系统容量和频谱效率，特别适合物联网和智能家居等低功耗、大量终端的场景。

### BSS Color

BSS Color（基础服务集着色）是 Wi-Fi 6 引入的同频干扰识别机制。AP 在传输时携带 6 bit 颜色标识，终端可识别来自不同 BSS 的信号，从而在检测到同频干扰时更快切换到空闲信道。

### 目标唤醒时间（TWT）

TWT（Target Wake Time）允许终端与 AP 协商精确的唤醒时间表，终端可在大部分时间处于低功耗休眠状态。TWT 技术对于电池供电的 IoT 设备尤为关键，可将功耗降低高达 90%。

## Wi-Fi Manager (wifi_mgmr) API

Wi-Fi Manager（wifi_mgmr）是 BL616/BL618 Wi-Fi 6 驱动的核心管理模块，负责无线连接、扫描、状态管理和 AP 控制等操作。

### 头文件

```c
#include "wifi_mgmr.h"
#include "wifi_mgmr_ext.h"
```

### 初始化

```c
int wifi_mgmr_init(void);
int wifi_mgmr_task_start(void);
```

`wifi_mgmr_init()` 初始化 Wi-Fi Manager 系统；`wifi_mgmr_task_start()` 启动 Wi-Fi 管理任务，开始处理无线事件。

### 连接参数

```c
typedef struct wifi_mgmr_sta_connect_params {
    char ssid[MGMR_SSID_LEN];          // SSID，最大 32 字节
    uint8_t ssid_len;                   // SSID 长度
    char key[MGMR_KEY_LEN];             // 密码，最大 64 字节
    uint8_t key_len;                     // 密码长度
    char bssid_str[MGMR_BSSID_LEN];     // BSSID（可选）
    char akm_str[MGMR_AKM_LEN];         // AKM 套件
    uint8_t pmf_cfg;                     // PMF 配置
    uint16_t freq1;                      // 主频率
    uint16_t freq2;                      // 次频率（可选）
    uint8_t use_dhcp;                    // 是否使用 DHCP
    uint8_t listen_interval;             // 监听间隔 [1, 100]
    uint8_t scan_mode;                   // 扫描模式
    uint8_t quick_connect;               // 快速连接标志
    int timeout_ms;                       // 超时时间（毫秒）
    uint16_t duration;                   // 扫描持续时间（TU）
    uint16_t probe_cnt;                  // 探针数量
    uint8_t auth_timeout;                 // 认证超时（秒）
} wifi_mgmr_sta_connect_params_t;
```

### STA 连接

```c
int wifi_mgmr_sta_connect(const wifi_mgmr_sta_connect_params_t *config);
```

建立与 AP 的连接。需预先填充连接参数结构体，包括 SSID、密码、频道等。

**示例：Wi-Fi 6 STA 连接**

```c
#include "wifi_mgmr.h"
#include "wifi_mgmr_ext.h"

void wifi6_sta_connect_example(void)
{
    wifi_mgmr_sta_connect_params_t conn_params = {0};

    // 配置 SSID
    memcpy(conn_params.ssid, "WiFi6_AP", 9);
    conn_params.ssid_len = 9;

    // 配置密码
    memcpy(conn_params.key, "password123", 11);
    conn_params.key_len = 11;

    // 配置 BSSID（可选）
    memcpy(conn_params.bssid_str, "aa:bb:cc:dd:ee:ff", 17);
    conn_params.bssid_str[17] = '\0';

    // 配置 5GHz 频段
    conn_params.freq1 = 5180;  // 36 频道

    // 启用 DHCP
    conn_params.use_dhcp = 1;

    // 连接超时 30 秒
    conn_params.timeout_ms = 30000;

    // 启动连接
    int ret = wifi_mgmr_sta_connect(&conn_params);
    if (ret == 0) {
        printf("Wi-Fi 6 连接请求已发送\r\n");
    }
}
```

### STA 断开连接

```c
void wifi_mgmr_sta_disconnect(void);
```

主动断开当前 STA 连接。

### 扫描 API

```c
int wifi_mgmr_sta_scan(const wifi_mgmr_scan_params_t *config);
int wifi_mgmr_scan_beacon_save(wifi_mgmr_scan_item_t *scan);
int wifi_mgmr_scan_ap_all(void *env, void *arg, scan_item_cb_t cb);
```

扫描参数结构体定义：

```c
typedef struct wifi_mgmr_scan_params {
    uint8_t ssid_length;
    uint8_t ssid_array[MGMR_SSID_LEN];  // 指定 SSID
    uint8_t bssid[6];                     // 指定 BSSID
    uint8_t bssid_set_flag;               // BSSID 设置标志
    uint8_t probe_cnt;                    // 探针请求数量
    int channels_cnt;                      // 频道数量
    uint8_t channels[MAX_FIXED_CHANNELS_LIMIT]; // 频道列表
    uint32_t duration;                     // 扫描持续时间
    bool passive;                          // 被动扫描模式
} wifi_mgmr_scan_params_t;
```

扫描结果项结构体：

```c
typedef struct wifi_mgmr_scan_item {
    uint32_t mode;
    uint32_t timestamp_lastseen;
    int ssid_len;
    uint8_t channel;
    int8_t rssi;               // 信号强度 (dBm)
    char ssid[32];             // SSID
    uint8_t bssid[6];          // BSSID
    int8_t ppm_abs;            // 绝对功率偏移
    int8_t ppm_rel;            // 相对功率偏移
    uint8_t auth;              // 认证类型
    uint8_t cipher;            // 加密类型
    uint8_t is_used;           // 是否已使用
    uint8_t wps;               // WPS 支持
    uint8_t best_antenna;      // 最佳天线
} wifi_mgmr_scan_item_t;
```

**示例：全频道主动扫描**

```c
void wifi6_scan_all_channels_example(void)
{
    wifi_mgmr_scan_params_t scan_params = {0};

    scan_params.ssid_length = 0;        // 扫描所有 SSID
    scan_params.bssid_set_flag = 0;     // 不指定 BSSID
    scan_params.probe_cnt = 3;           // 每个频道 3 次探针
    scan_params.passive = false;         // 主动扫描
    scan_params.duration = 100;          // 扫描持续 100 TU

    int ret = wifi_mgmr_sta_scan(&scan_params);
    if (ret == 0) {
        printf("扫描已启动\r\n");
    }
}
```

### 状态查询

```c
int wifi_mgmr_state_get(void);
int wifi_mgmr_sta_connect_ind_stat_get(wifi_mgmr_connect_ind_stat_info_t *info);
```

连接状态信息结构体：

```c
typedef struct wifi_mgmr_connect_ind_stat_info {
    uint16_t status_code;    // 状态码
    uint16_t reason_code;    // 原因码
    char ssid[33];           // 当前 SSID
    char passphr[65];        // 密码
    uint8_t bssid[6];        // BSSID
    uint8_t channel;         // 频道
    uint8_t security;        // 安全类型
    uint16_t aid;            // 关联 ID
    uint8_t vif_idx;         // 虚拟接口索引
    uint8_t ap_idx;          // AP 索引
    uint8_t ch_idx;          // 频道索引
    bool qos;                // QoS 支持
    uint8_t bss_mode;        // BSS 模式
} wifi_mgmr_connect_ind_stat_info_t;
```

### 国家码配置

```c
int wifi_mgmr_get_channel_nums(const char *country_code, uint8_t *c24G_cnt, uint8_t *c5G_cnt);
void wifi_mgmr_print_channel_info(const char *country_code);
```

**Wi-Fi 事件定义（wifi_mgmr_ext.h）**

```c
#define EV_WIFI                   0x0002
#define CODE_WIFI_ON_INIT_DONE    1   // 初始化完成
#define CODE_WIFI_ON_MGMR_DONE    2   // Manager 就绪
#define CODE_WIFI_CMD_RECONNECT   3   // 重连命令
#define CODE_WIFI_ON_CONNECTED    4   // 连接成功
#define CODE_WIFI_ON_DISCONNECT   5   // 断开连接
#define CODE_WIFI_ON_PRE_GOT_IP   6   // 获取 IP 前
#define CODE_WIFI_ON_GOT_IP       7   // 获取 IP 成功
#define CODE_WIFI_ON_CONNECTING   8   // 连接中
#define CODE_WIFI_ON_SCAN_DONE    9   // 扫描完成
#define CODE_WIFI_ON_AP_STARTED   11  // AP 已启动
#define CODE_WIFI_ON_AP_STOPPED   12  // AP 已停止
#define CODE_WIFI_ON_GOT_IP6      25  // 获取 IPv6
```

### TWT（目标唤醒时间）配置

Wi-Fi 6 支持 TWT 功能，允许设备与 AP 协商精确的唤醒时间。

```c
void cmd_wifi_mgmr_sta_twt_setup(int argc, char **argv);
void cmd_wifi_mgmr_sta_twt_teardown(int argc, char **argv);
void cmd_wifi_mgmr_sta_twt_statusget(int argc, char **argv);
```

**TWT 参数结构体（cfgmacsw.h）**

```c
struct cfgmacsw_twt_setup_req {
    uint16_t fhost_vif_idx;       // 虚拟接口索引
    uint8_t setup_type;            // 设置类型
    uint8_t flow_type;             // 流类型 (0: Announced, 1: Unannounced)
    uint8_t wake_int_exp;         // 唤醒间隔指数
    bool wake_dur_unit;            // 唤醒持续时间单位
    uint8_t min_twt_wake_dur;      // 最小唤醒持续时间
    uint16_t wake_int_mantissa;   // 唤醒间隔尾数
};
```

**示例：TWT 设置**

```c
void wifi6_twt_setup_example(void)
{
    // TWT 参数
    // wake_int_exp = 10, wake_int_mantissa = 1000
    // 实际唤醒间隔 = 2^10 * 1000 = 1024000 us = 1024 ms

    printf("设置 TWT: 唤醒间隔 1024ms\r\n");
    // 调用 CLI 命令或直接调用驱动 API
    // wifi_mgmr_sta_twt_setup(...);
}
```

### 功率控制

```c
int wifi_mgmr_ps_on(void);   // 启用功率节省
int wifi_mgmr_ps_off(void);  // 关闭功率节省
int wifi_mgmr_ps_set(int level);  // 设置功率等级
```

### AP 模式

```c
int wifi_mgmr_ap_start(wifi_mgmr_ap_params_t *params);
int wifi_mgmr_ap_stop(void);
int wifi_mgmr_ap_sta_list_get(void);
int wifi_mgmr_ap_sta_delete(uint8_t sta_idx);
```

AP 参数结构体：

```c
typedef struct wifi_mgmr_ap_params {
    char *ssid;                    // SSID
    char *key;                     // 密码
    char *akm;                     // AKM 套件 (OPEN/WPA/WPA2)
    uint8_t channel;               // 频道
    uint8_t type;                  // 信道带宽类型
    bool use_dhcpd;                // 启用 DHCP 服务
    int start;                      // DHCP 池起始地址
    int limit;                      // DHCP 池大小
    uint32_t ap_ipaddr;            // AP IP 地址
    uint32_t ap_mask;              // 子网掩码
    bool hidden_ssid;              // 隐藏 SSID
    bool isolation;                // 客户端隔离
    int bcn_interval;              // 信标间隔 (TU)
    uint8_t bcn_mode;              // 信标模式
    int bcn_timer;                 // 信标定时器
    bool disable_wmm;              // 禁用 WMM
} wifi_mgmr_ap_params_t;
```

## wifi6_lwip_adapter 与 MAT 适配层

`wifi6_lwip_adapter` 是连接 Wi-Fi 6 驱动与 LWIP 网络协议栈的适配层模块。核心功能包括网络接口管理、数据包收发、MAT（MAC Address Translation）等。

### MAT 模块

MAT（MAC Address Translation，MAC 地址转换）模块负责维护 IP 地址与 MAC 地址的映射关系，支持 ARP 代理和 ND（Neighbor Discovery）优化。

**头文件**

```c
#include "mat.h"
```

**MAT 错误码**

```c
#define MAT_ERR_OK      0   // 成功
#define MAT_ERR_INVAL   -1  // 无效参数
#define MAT_ERR_STATUS  -2  // 状态错误
#define MAT_ERR_MEM     -3  // 内存错误
#define MAT_ERR_DATA    -4  // 数据错误
#define MAT_ERR_PROTO   -5  // 协议错误
```

**MAT 元组结构体**

```c
struct mat_tuple {
    uint8_t hwaddr[6];     // MAC 地址
    uint8_t used;          // 使用标志
    uint32_t ts;           // 时间戳
    ip_addr_t ipaddr;      // IP 地址
};
```

**MAT API**

```c
int mat_tuple_add(uint8_t *hwaddr, ip_addr_t *ip);
// 添加 IP-MAC 映射元组

int mat_tuple_del(uint8_t *hwaddr, ip_addr_t *ip);
// 删除 IP-MAC 映射元组

int mat_handle_egress(struct netif *netif, struct pbuf *pbuf, struct pbuf **out);
// 处理出口数据包（发送前）

int mat_handle_ingress(struct netif *netif, struct pbuf *pbuf);
// 处理入口数据包（接收后）

struct mat_tuple *mat_tuple_find(uint8_t *hwaddr, ip_addr_t *ip);
// 查找 IP-MAC 映射元组
```

**MAT 工作原理**

MAT 模块在出口方向检查数据包目的 IP 是否已有对应的 MAC 映射：若有则直接封装；若没有则触发 ARP 学习过程。在入口方向，MAT 记录源 IP-MAC 映射以供后续使用。

### 网络接口定义（net_def.h）

```c
typedef struct netif        inet_if_t;      // 网络接口
typedef struct pbuf_custom  inet_buf_rx_t;  // 接收缓冲区
typedef struct pbuf         inet_buf_tx_t;  // 发送缓冲区

#define NET_AL_MAX_IFNAME   4               // 接口名最大长度
```

### LWIP 集成

wifi6_lwip_adapter 模块将 Wi-Fi 6 驱动封装为 LWIP 能够调用的 netif 接口，实现 IP 层的透明传输。应用程序无需关心底层无线驱动细节，可直接使用标准 LWIP socket API 进行网络通信。

**典型初始化流程**

```c
#include "wifi_mgmr.h"
#include "lwip/netif.h"

extern struct netif *netif_get_sta(void);

void wifi6_lwip_init_example(void)
{
    struct netif *sta_netif = netif_get_sta();

    // Wi-Fi 初始化
    wifi_mgmr_init();
    wifi_mgmr_task_start();

    // 连接 Wi-Fi
    wifi_mgmr_sta_connect(&conn_params);

    // LWIP 自动获取 IP（DHCP）
    // 应用程序可直接使用 socket API
}
```

## OFDMA 配置说明

OFDMA 是 Wi-Fi 6 的核心技术之一，BL616/BL618 驱动支持通过命令行或 API 配置 OFDMA 参数。

### OFDMA 参数

OFDMA 配置主要涉及信道带宽和资源单元划分。BL616/BL618 支持 20MHz、40MHz、80MHz 和 160MHz 信道带宽，对应不同的 RU（Resource Unit）划分方式。

### 信道带宽与 RU 配置

| 信道带宽 | 最大 RU 数 | 最小 RU 大小 |
|----------|------------|--------------|
| 20 MHz   | 9          | 26-tone     |
| 40 MHz   | 18         | 26-tone     |
| 80 MHz   | 37         | 26-tone     |
| 160 MHz  | 74         | 26-tone     |

### CLI 命令配置

Wi-Fi 6 OFDMA 可通过 shell 命令配置：

```bash
# 启用 Wi-Fi 6 模式
wifi_mode_set ax

# 配置 OFDMA 参数
ofdma_config 1  # 启用 OFDMA

# 查看 OFDMA 状态
ofdma_status
```

### 代码中配置 OFDMA

```c
// OFDMA 配置示例
void wifi6_ofdma_config_example(void)
{
    // 设置信道带宽为 80MHz
    uint8_t channel_width = 2;  // 0: 20MHz, 1: 40MHz, 2: 80MHz, 3: 160MHz

    // 启用 MU-MIMO
    uint8_t mu_mimo_enable = 1;

    // 启用 OFDMA
    uint8_t ofdma_enable = 1;

    printf("Wi-Fi 6 OFDMA 配置完成\r\n");
    printf("信道带宽: %s\r\n",
           channel_width == 2 ? "80MHz" : "其他");
    printf("MU-MIMO: %s\r\n",
           mu_mimo_enable ? "启用" : "禁用");
    printf("OFDMA: %s\r\n",
           ofdma_enable ? "启用" : "禁用");
}
```

## 常见问题与排查

### 连接失败

1. **检查 SSID 和密码**：确保配置正确
2. **检查频道**：确认设备支持的频道与 AP 一致
3. **检查信号强度**：RSSI 低于 -80 dBm 可能导致连接不稳定
4. **查看日志**：通过串口查看 Wi-Fi 事件码输出

### 扫描无结果

1. 确认 Wi-Fi 驱动已初始化
2. 检查国家码设置，确保所需频道可用
3. 尝试被动扫描模式（`passive = true`）

### TWT 不生效

1. 确认 AP 支持 TWT
2. 检查 TWT 参数配置是否合理
3. 确认固件支持 TWT 功能

## 参考

- [1] IEEE 802.11ax-2021 - IEEE Standard for Information Technology—Telecommunications and Information Exchange Between Systems—Local and Metropolitan Area Networks—Specific Requirements - Part 11: Wireless LAN Medium Access Control (MAC) and Physical Layer (PHY) Specifications - Amendment 1: Enhancements for High Efficiency WLAN
- [2] Bouffalo SDK Wi-Fi 6 Component Source: `/components/wireless/wifi6/fhost/include/wifi_mgmr.h`
- [3] Bouffalo SDK Wi-Fi 6 CLI Interface: `/components/wireless/wifi6/fhost/include/wifi_mgmr_cli.h`
- [4] Bouffalo SDK Wi-Fi 6 Extended API: `/components/wireless/wifi6/fhost/include/wifi_mgmr_ext.h`
- [5] Bouffalo SDK Network Definitions: `/components/wireless/wifi6/fhost/include/net_def.h`
- [6] Bouffalo SDK MAT (MAC Address Translation) Module: `/components/wireless/wifi6/wifi6_lwip_adapter/include/mat.h`
- [7] Bouffalo SDK CFG MACSW Interface: `/components/wireless/wifi6/fhost/include/cfgmacsw.h`
- [8] Wi-Fi 6 Technology Overview - Wi-Fi Alliance
- [9] LWIP - Lightweight IP Stack Documentation
