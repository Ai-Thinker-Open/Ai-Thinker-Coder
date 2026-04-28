# MACSW (MAC Software) 技术文档

## 概述

MACSW（MAC Software）是 Bouffalo Lab 无线 SDK 中的核心软件层，位于 Host CPU 与无线基带（Modem）之间，负责处理 IEEE 802.11 MAC 层的所有软件逻辑。作为连接高层网络协议栈与底层硬件的桥梁，MACSW 提供了帧收发、加密引擎管理、电源管理等关键功能。

MACSW 的设计目标是实现一个与硬件解耦的 MAC 层软件框架，使得上层协议栈（如 Wi-Fi Mgmr、wl80211）可以通过统一接口控制无线硬件，而无需关心具体的硬件实现细节。

### 核心特性

- **版本**: v6.10.0.0
- **协议支持**: 802.11a/b/g/n/ac/ax（Wi-Fi 6）
- **多接口支持**: 最多 4 个虚拟接口（VIF）
- **安全引擎**: 硬件 AES-CCMP/TKIP 加速
- **电源管理**: 支持 Legacy PS、UAPSD、TWT

---

## 架构层次

```
┌─────────────────────────────────┐
│     上层应用 / 网络协议栈         │
│   (wifi_mgmr / wl80211 / TCPIP) │
└───────────────┬─────────────────┘
                │
┌───────────────▼─────────────────┐
│         MACSW 层                │
│   (MAC Software - 本文档)       │
│  - 帧处理                      │
│  - 加密引擎管理                  │
│  - 电源管理                    │
│  - MIMO/MU-MIMO 控制            │
└───────────────┬─────────────────┘
                │
┌───────────────▼─────────────────┐
│       MAC HW / Modem            │
│   (无线基带硬件抽象层)           │
└─────────────────────────────────┘
```

在软件架构中，MACSW 处于比 wifi_mgmr 和 wl80211 更底层的位置。wifi_mgmr 负责 Wi-Fi 连接管理（如扫描、认证、关联），而 MACSW 直接与硬件交互，处理 MAC 层的具体实现。

---

## 头文件概览

| 头文件 | 功能描述 |
|--------|----------|
| `macsw.h` | 主头文件，包含版本、配置宏、核心 API 声明 |
| `macsw_plat.h` | 平台相关接口（初始化、帧收发、RTOS 任务管理） |
| `ieee80211.h` | IEEE 802.11 帧格式定义（Frame Control、Address、Sequence 等） |
| `mac_types.h` | MAC 层数据类型定义（帧类型、加密套件、速率集等） |
| `macsw_bridge_config.h` | Wi-Fi 桥接配置参数 |
| `bl_fw_api.h` | 固件 API 封装 |

---

## macsw_plat.h - 平台抽象接口

`macsw_plat.h` 定义了 MACSW 与底层硬件及 RTOS 交互的核心接口，包括初始化、帧收发、任务管理等功能。

### 核心函数

#### macsw_init()

初始化 MACSW 软件层。在系统启动时调用，用于初始化内部数据结构、配置硬件参数、创建 RTOS 任务。

#### macsw_send_frame()

发送原始 802.11 帧到无线信道。该函数接收完整的 MAC 帧数据，执行必要的硬件描述符填充后，将帧发送至空中接口。

#### macsw_recv_frame()

从无线信道接收 802.11 帧。当硬件检测到有效帧时，通过此函数将数据传递给上层协议栈处理。

### RTOS 任务管理

```c
// 创建 Wi-Fi 任务
void wifi_task_create(void);

// 暂停 Wi-Fi 任务
void wifi_task_suspend(void);

// 恢复 Wi-Fi 任务（通常从中断上下文调用）
void wifi_task_resume(bool isr);

// 获取系统运行时间（毫秒）
uint32_t wifi_sys_now_ms(bool isr);
```

### 中断管理

MACSW 使用临界区保护共享资源：

```c
#define GLOBAL_INT_DISABLE()      // 禁用全局中断
#define GLOBAL_INT_RESTORE()      // 恢复全局中断
```

### 日志接口

```c
void wifi_syslog(int priority, const char *fmt, ...);
```

---

## ieee80211.h - 802.11 帧格式定义

`ieee80211.h` 定义了 IEEE 802.11 标准规定的帧结构，包括 Frame Control、Duration、Address、Sequence Control、QoS Control 等字段。

### Frame Control 字段

```c
#define WIRELESS_80211_FCTL_FTYPE     0x000c  // 帧类型掩码
#define WIRELESS_80211_FCTL_STYPE     0x00f0  // 子类型掩码

#define WIRELESS_80211_FTYPE_MGMT     0x0000  // 管理帧
#define WIRELESS_80211_FTYPE_DATA     0x0008  // 数据帧

/* 管理帧子类型 */
#define WIRELESS_80211_STYPE_PROBE_REQ    0x0040  // 探测请求
#define WIRELESS_80211_STYPE_PROBE_RESP   0x0050  // 探测响应
#define WIRELESS_80211_STYPE_BEACON       0x0080  // 信标帧
#define WIRELESS_80211_STYPE_DISASSOC     0x00A0  // 解除关联
#define WIRELESS_80211_STYPE_AUTH         0x00B0  // 认证
#define WIRELESS_80211_STYPE_DEAUTH       0x00C0  // 解除认证
#define WIRELESS_80211_STYPE_ACTION       0x00D0  // Action 帧

/* 数据帧子类型 */
#define WIRELESS_80211_STYPE_QOS_DATA     0x0080  // QoS 数据帧
```

### 帧类型判断辅助函数

```c
// 判断是否为信标帧
static inline bool wireless_80211_is_beacon(__le16 fc);

// 判断是否为解除认证帧
static inline bool wireless_80211_is_deauth(__le16 fc);

// 判断是否为解除关联帧
static inline bool wireless_80211_is_disassoc(__le16 fc);

// 判断是否为 Action 帧
static inline bool wireless_80211_is_action(__le16 fc);

// 判断是否为探测响应帧
static inline int wireless_80211_is_probe_resp(__le16 fc);

// 判断是否为探测请求帧
static inline bool wireless_80211_is_probe_req(__le16 fc);

// 判断是否为数据帧
static inline bool wireless_80211_is_data(__le16 fc);

// 判断是否为 QoS 数据帧
static inline bool wireless_80211_is_data_qos(__le16 fc);
```

### Sequence Control

```c
#define WIRELESS_80211_SCTL_SEQ         0xFFF0  // 序列号掩码
#define WIRELESS_80211_SN_MASK          ((WIRELESS_80211_SCTL_SEQ) >> 4)
#define WIRELESS_80211_MAX_SN           WIRELESS_80211_SN_MASK
#define WIRELESS_80211_SN_MODULO       (WIRELESS_80211_MAX_SN + 1)
```

### 802.11 Reason Codes

```c
enum wireless_80211_reasoncode {
    WIRELESS_REASONCODE_UNSPECIFIED = 1,
    WIRELESS_REASONCODE_PRE_AUTH_NOT_VALID = 2,
    WIRELESS_REASONCODE_DEAUTH_LEAVING = 3,
    WIRELESS_REASONCODE_DISASSOC_DUE_TO_INACTIVITY = 4,
    WIRELESS_REASONCODE_DISASSOC_AP_BUSY = 5,
    WIRELESS_REASONCODE_CLASS2_FRAME_FROM_NONAUTH_STA = 6,
    WIRELESS_REASONCODE_CLASS3_FRAME_FROM_NONASSOC_STA = 7,
    WIRELESS_REASONCODE_DISASSOC_STA_HAS_LEFT = 8,
    WIRELESS_REASONCODE_STA_REQ_ASSOC_WITHOUT_AUTH = 9,
};
```

---

## mac_types.h - MAC 层数据类型

`mac_types.h` 定义了 MACSW 使用的所有核心数据结构，包括接口类型、MAC 地址、SSID、通道定义、加密套件、速率集等。

### 虚拟接口类型

```c
enum mac_vif_type {
    VIF_STA,           // ESS STA 接口
    VIF_IBSS,          // IBSS STA 接口
    VIF_AP,            // AP 接口
    VIF_MESH_POINT,    // Mesh Point 接口
    VIF_MONITOR,       // 监听接口
    VIF_UNKNOWN
};
```

### MAC 地址

```c
#define MAC_ADDR_LEN 6

struct mac_addr {
    uint16_t array[MAC_ADDR_LEN/2];  // 3 个 16 位字组成 6 字节地址
};
```

### SSID

```c
#define MAC_SSID_LEN 32

struct mac_ssid {
    uint8_t length;                  // SSID 实际长度
    uint8_t array[MAC_SSID_LEN];      // SSID 字符数组
};
```

### 通道定义

```c
// 频段
enum mac_chan_band {
    PHY_BAND_2G4 = 0,    // 2.4GHz 频段
    PHY_BAND_5G,         // 5GHz 频段
    PHY_BAND_MAX
};

// 带宽
enum mac_chan_bandwidth {
    PHY_CHNL_BW_20,      // 20MHz
    PHY_CHNL_BW_40,      // 40MHz
    PHY_CHNL_BW_80,      // 80MHz
    PHY_CHNL_BW_160,     // 160MHz
    PHY_CHNL_BW_80P80,   // 80+80MHz
    PHY_CHNL_BW_OTHER
};

// 通道标志
enum mac_chan_flags {
    CHAN_NO_IR = CO_BIT(0),        // 禁止发射
    CHAN_DISABLED = CO_BIT(1),     // 通道禁用
    CHAN_RADAR = CO_BIT(2),        // 需要雷达检测
    CHAN_DISABLE_VHT = CO_BIT(6),  // 禁用 VHT
    CHAN_DISABLE_HE = CO_BIT(7),   // 禁用 HE
};

// 通道定义
struct mac_chan_def {
    uint16_t freq;                 // 频率（MHz）
    uint8_t band;                  // 频段
    uint8_t flags;                  // 标志
    int8_t tx_power;               // 最大发射功率（dBm）
};

// 通道操作参数
struct mac_chan_op {
    uint8_t band;
    uint8_t type;                  // 带宽类型
    uint16_t prim20_freq;          // 主 20MHz 频率
    uint16_t center1_freq;         // 中心频率 1
    uint16_t center2_freq;         // 中心频率 2
    int8_t tx_power;
    uint8_t flags;
};
```

### 加密套件

```c
enum mac_cipher_suite {
    MAC_CIPHER_WEP40 = 1,          // WEP-40
    MAC_CIPHER_TKIP = 2,           // TKIP
    MAC_CIPHER_CCMP = 4,           // CCMP-128
    MAC_CIPHER_WEP104 = 5,         // WEP-104
    MAC_CIPHER_WPI_SMS4 = 6,       // WAPI
    MAC_CIPHER_BIP_CMAC_128 = 6,  // AES_CMAC
    MAC_CIPHER_GCMP_128 = 8,       // GCMP-128
    MAC_CIPHER_GCMP_256 = 9,      // GCMP-256
    MAC_CIPHER_CCMP_256 = 10,      // CCMP-256
    MAC_CIPHER_BIP_GMAC_128 = 11,
    MAC_CIPHER_BIP_GMAC_256 = 12,
    MAC_CIPHER_BIP_CMAC_256 = 13,
    MAC_CIPHER_INVALID = 0xFF
};
```

### 认证与密钥管理（AKM）

```c
enum mac_akm_suite {
    MAC_AKM_NONE = 0,               // 无安全
    MAC_AKM_PRE_RSN = 1,            // Pre-RSNA (WEP/WPA)
    MAC_AKM_8021X = 2,             // 802.1X
    MAC_AKM_PSK = 3,               // PSK
    MAC_AKM_FT_8021X = 4,          // FT 802.1X
    MAC_AKM_FT_PSK = 5,            // FT PSK
    MAC_AKM_8021X_SHA256 = 6,      // 802.1X + SHA256
    MAC_AKM_PSK_SHA256 = 7,        // PSK + SHA256
    MAC_AKM_TDLS = 8,              // TDLS
    MAC_AKM_SAE = 9,               // SAE
    MAC_AKM_FT_OVER_SAE = 10,      // FT over SAE
    MAC_AKM_8021X_SUITE_B = 11,    // 802.1X Suite B
    MAC_AKM_8021X_SUITE_B_192 = 12,
    MAC_AKM_FILS_SHA256 = 14,      // FILS + SHA256
    MAC_AKM_FILS_SHA384 = 15,      // FILS + SHA384
    MAC_AKM_FT_FILS_SHA256 = 16,
    MAC_AKM_FT_FILS_SHA384 = 17,
    MAC_AKM_OWE = 18,              // OWE
    MAC_AKM_WAPI_CERT = 256,       // WAPI 证书
    MAC_AKM_WAPI_PSK = 257,        // WAPI PSK
};
```

### Wi-Fi 模式

```c
typedef enum {
    WIFI_MODE_802_11B = 0x01,       // 802.11b
    WIFI_MODE_802_11A = 0x02,       // 802.11a
    WIFI_MODE_802_11G = 0x04,       // 802.11g
    WIFI_MODE_802_11N_2_4 = 0x08,  // 802.11n @ 2.4GHz
    WIFI_MODE_802_11N_5 = 0x10,     // 802.11n @ 5GHz
    WIFI_MODE_802_11AC_5 = 0x20,   // 802.11ac @ 5GHz
    WIFI_MODE_802_11AC_2_4 = 0x40, // 802.11ac @ 2.4GHz
    WIFI_MODE_802_11AX_2_4 = 0x80, // 802.11ax @ 2.4GHz (Wi-Fi 6)
    WIFI_MODE_802_11AX_5 = 0x100,  // 802.11ax @ 5GHz (Wi-Fi 6)
} WiFi_Mode_t;

// 常用组合模式
#define WIFI_MODE_BGN (WIFI_MODE_802_11B | WIFI_MODE_802_11G | WIFI_MODE_802_11N_2_4)
#define WIFI_MODE_BGNAX (WIFI_MODE_802_11B | WIFI_MODE_802_11G | WIFI_MODE_802_11N_2_4 | WIFI_MODE_802_11AX_2_4)
```

### 访问类别与 TID

```c
enum mac_ac {
    AC_BK = 0,      // Background
    AC_BE = 1,      // Best-effort
    AC_VI = 2,      // Video
    AC_VO = 3,      // Voice
    AC_MAX
};

enum mac_tid {
    TID_0 = 0, TID_1, TID_2, TID_3, TID_4, TID_5, TID_6, TID_7,
    TID_MGT,        // 管理 TID
    TID_MAX
};
```

### 速率定义

```c
enum mac_legacy_rates {
    MAC_RATE_1MBPS = 2,
    MAC_RATE_2MBPS = 4,
    MAC_RATE_5_5MBPS = 11,
    MAC_RATE_11MBPS = 22,
    MAC_RATE_6MBPS = 12,
    MAC_RATE_9MBPS = 18,
    MAC_RATE_12MBPS = 24,
    MAC_RATE_18MBPS = 36,
    MAC_RATE_24MBPS = 48,
    MAC_RATE_36MBPS = 72,
    MAC_RATE_48MBPS = 96,
    MAC_RATE_54MBPS = 108
};

struct mac_rateset {
    uint8_t length;                  // 速率数量
    uint8_t array[MAC_RATESET_LEN];  // 速率数组
};
```

### 安全密钥

```c
#define MAC_SEC_KEY_LEN 32

struct mac_sec_key {
    uint8_t length;                  // 密钥长度
    uint32_t array[MAC_SEC_KEY_LEN/4]; // 密钥数据
};
```

### 扫描结果

```c
struct mac_scan_result {
    bool valid_flag;                  // 结果有效标志
    struct mac_addr bssid;           // BSSID
    struct mac_ssid ssid;            // SSID
    uint16_t bsstype;                // BSS 类型
    struct mac_chan_def *chan;       // 通道信息
    uint32_t akm;                    // 支持的 AKM 套件
    uint16_t group_cipher;           // 组加密套件
    uint16_t pairwise_cipher;        // 单播加密套件
    int8_t rssi;                     // 信号强度
    uint8_t multi_bssid_index;
    uint8_t max_bssid_indicator;
    bool ftm_support;                // FTM 支持
    void *rxu_mgmt_ind;
};
```

---

## macsw_bridge_config.h - 桥接配置

`macsw_bridge_config.h` 定义了 Wi-Fi 桥接模式的相关配置参数，包括收发描述符数量、重排序缓冲区大小等。

```c
#define CFG_BARX 12                  // RX Block Ack 请求数
#define CFG_BATX 12                  // TX Block Ack 请求数
#define CFG_REORD_BUF 12             // 重排序缓冲区数量

// A-MSDU 配置
#define CFG_AMSDU_8K                 // 启用 8K A-MSDU 支持

// TX 描述符配置
#define CFG_TXDESC0 12
#define CFG_TXDESC1 64
#define CFG_TXDESC2 12
#define CFG_TXDESC3 12
#define CFG_TXDESC4 2

// 接收缓冲区配置
#define CONFIG_FHOST_RX_BUF_SECTION ".psram_noinit"
```

---

## 加密引擎

MACSW 支持通过硬件加速实现多种加密套件，包括：

### AES-CCMP

CCMP（Counter Mode CBC-MAC Protocol）是 802.11i 标准的强制要求，用于保护 WPA2 网络的帧安全。

```c
// 获取 CCMP 加密状态
uint8_t inline_macsw_mac_ccmp_getf(void);
```

### TKIP

TKIP（Temporal Key Integrity Protocol）是 WPA 标准的加密协议，提供与 WEP 的向后兼容性。

```c
// 获取 TKIP 加密状态
uint8_t inline_macsw_mac_tkip_getf(void);
```

### GCMP

GCMP（Galois/Counter Mode Protocol）是 802.11ad/ax 标准的高效加密协议。

```c
// 获取 GCMP 加密状态
uint8_t inline_macsw_mac_gcmp_getf(void);
```

硬件加密引擎位于 MAC 层与 Modem 之间，当 MACSW 发送帧时，数据自动通过硬件加密模块进行加密处理；当接收帧时，同样自动解密后再交给软件处理。

---

## 与 wifi_mgmr / wl80211 的关系

MACSW 位于软件栈的更底层，直接与硬件交互：

```
┌─────────────────────────┐
│     wifi_mgmr            │  连接管理、扫描、认证、关联
├─────────────────────────┤
│     wl80211              │  802.11 驱动抽象层
├─────────────────────────┤
│     MACSW (本文档)        │  MAC 层软件实现
├─────────────────────────┤
│     MAC HW / Modem       │  硬件抽象层
└─────────────────────────┘
```

- **wifi_mgmr**: 负责 Wi-Fi 连接状态机，处理扫描结果、发起认证/关联请求
- **wl80211**: 提供类似 Linux nl80211 的接口，用于配置无线设备
- **MACSW**: 实际执行 MAC 层操作，包括帧格式化、硬件描述符填充、加密处理

---

## 代码示例

### MACSW 初始化

```c
#include "macsw.h"
#include "macsw_plat.h"

// 系统初始化时调用
void system_wifi_init(void)
{
    // 创建 Wi-Fi 任务
    wifi_task_create();
}

// 或者在独立初始化场景
void macsw_example_init(void)
{
    // 初始化 MACSW 组件
    // 初始化硬件描述符
    // 配置默认通道
    // 使能中断
}
```

### 发送原始 802.11 帧

```c
#include "macsw.h"
#include "ieee80211.h"

// 发送原始 802.11 数据帧示例
int send_raw_80211_frame(uint8_t *dest_mac, uint8_t *payload, uint16_t payload_len)
{
    // 帧格式：
    // [Frame Control(2)] [Duration(2)] [Address1(6)] [Address2(6)] [Address3(6)] [Sequence Control(2)] [payload]
    //
    // 对于数据帧：
    // - Address1: 接收方 MAC (RA)
    // - Address2: 发送方 MAC (TA)
    // - Address3: BSSID 或目的地址
    
    uint8_t frame[256];
    uint16_t frame_len = 0;
    __le16 fc;
    
    // 构造 Frame Control (数据帧, To DS)
    fc = WIRELESS_80211_FTYPE_DATA | WIRELESS_80211_STYPE_QOS_DATA;
    // 设置 To DS 位
    fc |= cpu_to_le16(0x0001);
    
    frame[0] = fc & 0xFF;
    frame[1] = (fc >> 8) & 0xFF;
    frame_len += 2;
    
    // Duration (16 bits)
    frame[frame_len++] = 0x00;
    frame[frame_len++] = 0x00;
    
    // Address1 (DA)
    memcpy(&frame[frame_len], dest_mac, 6);
    frame_len += 6;
    
    // Address2 (SA) - 本机 MAC
    uint8_t local_mac[6] = {0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF};
    memcpy(&frame[frame_len], local_mac, 6);
    frame_len += 6;
    
    // Address3 (BSSID)
    uint8_t bssid[6] = {0x11, 0x22, 0x33, 0x44, 0x55, 0x66};
    memcpy(&frame[frame_len], bssid, 6);
    frame_len += 6;
    
    // Sequence Control (12 bits sequence number + 4 bits fragment number)
    frame[frame_len++] = 0x00;
    frame[frame_len++] = 0x10;  // Sequence number = 1
    frame_len += 2;  // QoS Control
    
    // Payload
    memcpy(&frame[frame_len], payload, payload_len);
    frame_len += payload_len;
    
    // FCS (4 bytes) - 通常由硬件自动添加
    // frame[frame_len++] = 0x00;
    // frame[frame_len++] = 0x00;
    // frame[frame_len++] = 0x00;
    // frame[frame_len++] = 0x00;
    
    // 调用发送接口
    // macsw_send_frame(frame, frame_len);
    
    return 0;
}
```

### 解析 802.11 帧类型

```c
#include "ieee80211.h"

void handle_incoming_frame(uint8_t *frame, uint16_t frame_len)
{
    if (frame_len < 2) return;
    
    __le16 fc = frame[0] | ((__le16)frame[1] << 8);
    
    if (wireless_80211_is_beacon(fc)) {
        // 处理信标帧
        // printf("Received Beacon frame\n");
    } else if (wireless_80211_is_probe_req(fc)) {
        // 处理探测请求帧
        // printf("Received Probe Request\n");
    } else if (wireless_80211_is_data(fc)) {
        // 处理数据帧
        if (wireless_80211_is_data_qos(fc)) {
            // printf("Received QoS Data frame\n");
        } else {
            // printf("Received Data frame\n");
        }
    } else if (wireless_80211_is_deauth(fc)) {
        // 处理解除认证帧
        // printf("Received Deauth frame\n");
    }
}
```

---

## 配置宏参考

### 版本信息

```c
#define MACSW_VERSION_STR "v6.10.0.0"
#define MACSW_VERSION_MAJ 6
#define MACSW_VERSION_MIN 10
#define MACSW_VERSION_REL 0
#define MACSW_VERSION_PAT 0
```

### 功能开关

| 宏 | 说明 |
|----|------|
| `CFG_BCN` | 信标支持 |
| `CFG_PS` | 电源管理 |
| `CFG_UAPSD` | U-APSD 节能模式 |
| `CFG_MFP` | MFP (Management Frame Protection) |
| `CFG_AMSDU` | A-MSDU 聚合 |
| `CFG_AMPDU_TX` | A-MPDU 发送聚合 |
| `CFG_AMPDU_RX` | A-MPDU 接收聚合 |
| `CFG_VHT` | VHT (802.11ac) |
| `CFG_HE` | HE (802.11ax/Wi-Fi 6) |
| `CFG_MESH` | Mesh 网络支持 |
| `CFG_P2P` | P2P 支持 |

### 限制参数

```c
#define MACSW_VIRT_DEV_MAX 4      // 最大虚拟接口数
#define MACSW_REMOTE_STA_MAX     // 最大关联站点数
#define MACSW_MAX_BA_TX          // 最大 TX Block Ack 数
#define MACSW_MAX_BA_RX          // 最大 RX Block Ack 数
#define MACSW_HEAP_SIZE          // 堆内存大小
```

---

## 参考

- [Bouffalo SDK 文档](../README.md) - SDK 整体介绍
- [Wi-Fi 驱动组件文档](../wl80211/README.md) - wl80211 Wi-Fi 驱动详细说明
- IEEE Std 802.11™ - 802.11 协议标准
- Bouffalo Lab BL618 产品手册
