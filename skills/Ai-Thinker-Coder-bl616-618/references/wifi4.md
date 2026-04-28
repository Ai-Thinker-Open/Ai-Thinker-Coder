# Wi-Fi 4 (802.11n) 技术参考

## 概述

Wi-Fi 4 是基于 IEEE 802.11n 标准的无线网络协议，又称 HT（High Throughput）模式。该模式主要用于兼容老旧 802.11b/g 设备，或在 Wi-Fi 6（802.11ax）环境中作为降级模式使用。在 Bouffalo Lab SDK 中，Wi-Fi 4 由 `wifi4` 组件实现，支持 2.4GHz 和 5GHz 双频段工作。

802.11n 在物理层引入多项增强技术，显著提升了吞吐量与覆盖范围。相比 802.11g 的 54Mbps 理论速率，802.11n 在 40MHz 带宽和 Short GI 条件下可达 150Mbps（单空间流）。对于嵌入式设备而言，Wi-Fi 4 提供了良好的性价比——既满足主流 IoT 设备带宽需求，又保持较低的硬件成本和功耗。

## 802.11n 核心特性

### MIMO 1x1 与空间流

802.11n 引入多输入多输出（MIMO）技术，通过多天线实现空间复用。本芯片平台在 Wi-Fi 4 模式下仅使用单空间流（1x1 MIMO），即一根天线用于发送、一根用于接收。虽然无法利用多空间流提升峰值速率，但 MIMO 仍带来了分集增益，提升了信号稳定性与覆盖效果。

若系统配置有多根天线，可在 Wi-Fi 6 模式下启用 2x2 或更高阶 MIMO。Wi-Fi 4 的 1x1 约束使其更适合对功耗敏感的电池供电设备。

### Short GI（Short Guard Interval）

guard interval（保护间隔）是 OFDM 符号之间的时间缓冲，用于避免符号间干扰。标准 GI 为 800ns，Short GI 缩短至 400ns，使每个符号传输时间从 4μs 降至 3.6μs。Short GI 可将吞吐量提升约 11%，但对信道条件要求更高，适用于短距离、低干扰场景。

在 SDK 中通过 `bl_mod_params.sgi` 参数控制 Short GI 开关：

```c
// 启用 Short GI
bl_mod_params.sgi = true;

// 禁用 Short GI（使用标准 800ns GI）
bl_mod_params.sgi = false;
```

### 40MHz 带宽

802.11n 支持将两个 20MHz 信道捆绑为 40MHz 带宽使用，使子载波数量从 52 增加到 108，从而将单空间流速率从 72.2Mbps 提升至 150Mbps。40MHz 带宽仅在 2.4GHz 的非拥挤信道上推荐使用（避免与相邻 20MHz 频道冲突），在 5GHz 频段则更为常用。

40MHz 配置通过 `bl_mod_params.use_2040` 参数启用：

```c
// 启用 40MHz 带宽支持
bl_mod_params.use_2040 = true;
```

需要注意的是，40MHz 操作需要在 AP 和 STA 双方均支持的情况下协商成功才能生效。

### MCS 调制与编码方案

MCS（Modulation and Coding Scheme）索引定义了 802.11n 中的调制方式与码率组合。Wi-Fi 4 支持 MCS 0 至 MCS 15，每个索引对应特定的调制阶数和码率：

| MCS 索引 | 调制方式 | 码率 | 40MHz 单空间流速率 |
|---------|---------|------|-------------------|
| MCS 0   | BPSK    | 1/2  | 15 Mbps           |
| MCS 1   | QPSK    | 1/2  | 30 Mbps           |
| MCS 2   | QPSK    | 3/4  | 45 Mbps           |
| MCS 3   | 16-QAM  | 1/2  | 60 Mbps           |
| MCS 4   | 16-QAM  | 3/4  | 90 Mbps           |
| MCS 5   | 64-QAM  | 2/3  | 120 Mbps          |
| MCS 6   | 64-QAM  | 3/4  | 135 Mbps          |
| MCS 7   | 64-QAM  | 5/6  | 150 Mbps          |
| MCS 8   | 256-QAM | 3/4  | 180 Mbps (需芯片支持) |
| MCS 9   | 256-QAM | 5/6  | 200 Mbps (需芯片支持) |

默认 MCS 映射通过 `bl_mod_params.mcs_map` 配置。MCS 8 和 MCS 9 依赖 256-QAM 调制，仅在信噪比良好的环境下启用。

## bl_wifi_driver 架构

### LMAC 消息队列概述

Wi-Fi 驱动采用 LMAC（Lower MAC）架构，将协议栈分为 Host（驱动）侧和 Firmware（固件）侧。两者通过消息队列（Message Queue）进行通信，Host 侧发送请求消息（REQ），Firmware 侧返回确认消息（CFM）或异步指示消息（IND）。

消息结构定义在 `lmac_msg.h` 中：

```c
struct lmac_msg {
    ke_msg_id_t     id;         // 消息 ID
    ke_task_id_t    dest_id;    // 目标任务 ID
    ke_task_id_t    src_id;     // 源任务 ID
    u32             param_len;  // 参数长度
    u32             param[];    // 参数数组（字对齐）
};
```

消息 ID 由宏 `MSG_T(msg)` 和 `MSG_I(msg)` 解析，分别提取消息类型和实例编号。驱动任务 ID 预定义为 `DRV_TASK_ID (100)`。

### 关键消息类型

#### MM_START_REQ / MM_START_CFM

MAC 层启动请求，用于初始化 Wi-Fi 固件。包含 PHY 配置、超时参数和本地时钟精度：

```c
struct mm_start_req {
    struct phy_cfg_tag phy_cfg;     // PHY 配置
    u32_l              uapsd_timeout; // UAPSD 超时（毫秒）
    u16_l              lp_clk_accuracy; // LP 时钟精度（ppm）
};
```

#### MM_SET_CHANNEL_REQ

设置工作信道与带宽类型。信道类型定义包括 20MHz、40MHz、80MHz 等：

```c
struct mm_set_channel_req {
    u8_l    band;        // 频段：2.4GHz 或 5GHz
    u8_l    type;        // 信道类型：20/40/80/160MHz
    u16_l   prim20_freq; // 主 20MHz 信道频率（MHz）
    u16_l   center1_freq; // 中心频率 1
    u16_l   center2_freq; // 中心频率 2（用于 80+80）
    u8_l    index;       // RF 索引（主或副）
    s8_l    tx_power;    // 最大发射功率（dBm）
};
```

#### MM_SET_PS_MODE_REQ

电源管理模式切换，参数 `new_state` 可选 `MM_PS_MODE_OFF`、`MM_PS_MODE_ON` 或 `MM_PS_MODE_ON_DYN`（动态）。

#### SCAN_START_REQ / SCANU_START_REQ

发起网络扫描请求，支持多信道、多 SSID 被动或主动扫描：

```c
struct scan_start_req {
    struct scan_chan_tag chan[SCAN_CHANNEL_MAX]; // 信道列表
    struct mac_ssid      ssid[SCAN_SSID_MAX];    // SSID 列表
    struct mac_addr      bssid;                  // BSSID 过滤
    struct mac_addr      mac;                   // 发送源 MAC
    u32_l                add_ies;               // 额外 IEs 指针
    u16_l                add_ie_len;            // 额外 IEs 长度
    u8_l                 vif_idx;               // VIF 索引
    u8_l                 chan_cnt;              // 信道数量
    u8_l                 ssid_cnt;              // SSID 数量
    bool                 no_cck;                // 禁止 CCK 速率
};
```

### 接口类型

系统支持多种接口类型，通过 `mm_add_if_req` 结构中的 `type` 字段指定：

- `MM_STA`：基础服务集（ESS）Station 接口，最常用模式
- `MM_IBSS`：独立基础服务集（Ad-Hoc）接口
- `MM_AP`：接入点（Access Point）接口
- `MM_MESH_POINT`：Mesh 节点接口

## bl_mod_params_t 参数配置

`bl_mod_params` 结构体集中管理 Wi-Fi 4 的功能开关与性能参数，位于 `bl_mod_params.h`：

```c
struct bl_mod_params {
    bool ht_on;           // HT（802.11n）使能
    bool vht_on;          // VHT（802.11ac）使能（本芯片不支持）
    int  mcs_map;         // MCS 映射配置
    int  phy_cfg;         // PHY 配置索引
    int  uapsd_timeout;   // UAPSD 超时时间（毫秒）
    bool sgi;             // Short GI 使能
    bool sgi80;           // 80MHz Short GI 使能
    bool use_2040;        // 40MHz 带宽使能
    int  listen_itv;      // 监听间隔
    bool listen_bcmc;     // 监听广播/多播帧
    int  lp_clk_ppm;      // 低功耗时钟精度（ppm）
    bool ps_on;           // 电源管理使能
    int  tx_lft;          // 传输帧寿命（TUs）
    int  amsdu_maxnb;     // A-MSDU 最大数量
    int  uapsd_queues;    // UAPSD 队列掩码
};
```

### 功率配置

发射功率通过 `mm_set_power_req` 消息动态调整：

```c
struct mm_set_power_req {
    u8_l inst_nbr;  // 接口编号
    s8_l power;     // 发射功率（dBm）
};
```

典型值范围为 0dBm 至 20dBm，具体取决于硬件能力与法规限制。

### 速率配置

基础速率集通过 `mm_set_basic_rates_req` 设置，用于控制管理帧和广播帧的发送速率：

```c
struct mm_set_basic_rates_req {
    u32_l rates;     // 基础速率掩码（对应 bssBasicRateSet 寄存器）
    u8_l  inst_nbr;  // 接口编号
    u8_l  band;      // 频段
};
```

### 天线配置

天线选择在 PHY 配置中完成，支持多路径映射与分集。天线配置由 `phy_trd_cfg_tag` 或 `phy_karst_cfg_tag` 结构定义，包含 RF 路径映射和 IQ 补偿参数。

## Wi-Fi 4 节能机制

### APSD（Automatic Power Save Delivery）

U-APSD（Unscheduled Automatic Power Save Delivery）是 802.11n 推荐的节能机制，允许 AP 通过触发帧（Trigger Frame）唤醒 STA 接收下行业务。U-APSD 特别适用于 VoIP、视频流等对延迟敏感的应用场景。

APSD 配置通过以下参数控制：

- `bl_mod_params.uapsd_timeout`：U-APSD 超时时间，决定 STA 在无数据交互后返回休眠的等待时长
- `bl_mod_params.uapsd_queues`：使能 U-APSD 的 AC 队列掩码（bit0=VO, bit1=VI, bit2=BK, bit3=BE）
- `bl_mod_params.ps_on`：全局电源管理开关

```c
// 配置 U-APSD：使能 VO 和 VI 队列的自动节能
bl_mod_params.uapsd_queues = 0x03;  // VO(0x01) | VI(0x02)
bl_mod_params.uapsd_timeout = 300;  // 300ms 无数据后进入休眠
bl_mod_params.ps_on = true;
```

### 动态电源管理

`MM_PS_MODE_ON_DYN` 模式支持动态切换：STA 根据业务流量自动在激活与休眠状态间转换。当无数据到达时进入休眠以节省功耗，当检测到业务时快速唤醒。

监听间隔（Listen Interval）控制 STA 唤醒接收 Beacon 的频率：

```c
struct mm_set_ps_options_req {
    u8_l  vif_index;           // VIF 索引
    u16_l listen_interval;     // 监听间隔（Beacon 周期数，0 表示基于 DTIM）
    bool_l dont_listen_bc_mc;  // 是否忽略广播/多播
};
```

较长的监听间隔可降低功耗，但可能导致错过下行的广播/多播数据。

### 电源管理状态指示

Firmware 通过 `mm_ps_change_ind` 消息通知 Host 对端 STA 的电源状态变化：

```c
struct mm_ps_change_ind {
    u8_l sta_idx;     // 站点索引
    u8_l ps_state;    // 电源状态：0=激活，1=休眠
};
```

## Wi-Fi 4 与 Wi-Fi 6 的关系

Wi-Fi 6（802.11ax）在本平台 SDK 中以独立组件 `wifi6` 实现，而 `wifi4` 组件是整个无线栈的基础实现层。两者的关系体现在以下几个方面：

### 代码复用

Wi-Fi 4 的 LMAC 消息队列、扫描机制、电源管理框架被 Wi-Fi 6 直接复用。Wi-Fi 6 在此基础上增加了 OFDMA、1024-QAM、BSS Coloring 等新特性，但底层帧交换、PHY 配置流程与 Wi-Fi 4 保持一致。

### 协商降级

当 Wi-Fi 6 STA 关联到仅支持 Wi-Fi 4 的 AP 时，系统会自动协商至 802.11n 模式。反之，在高密度部署环境中，为兼容老旧设备或降低功耗，Wi-Fi 6 AP 可配置为仅允许 Wi-Fi 4 客户端关联。

### 参数继承

`bl_mod_params.ht_on` 控制 802.11n 功能，Wi-Fi 6 模式下应保持 `ht_on = true`（因为 802.11ax 需向后兼容 802.11n）。`mcs_map` 参数同样影响 Wi-Fi 6 的基础速率配置。

### 共存机制

Wi-Fi 4 和 Wi-Fi 6 可通过双频并发（Dual-Band Concurrent）实现共存：一个频段以 Wi-Fi 4 模式连接老旧设备，另一个频段以 Wi-Fi 6 提供高速接入。这需要 RF 硬件支持双前端。

## 代码示例

### Wi-Fi 4 模式连接示例

以下代码演示如何配置并启动 Wi-Fi 4（802.11n）连接：

```c
#include "bl_wifi_driver.h"
#include "bl_mod_params.h"
#include "lmac_msg.h"

// 初始化 Wi-Fi 4 参数
void wifi4_connect_init(void)
{
    // 使能 802.11n，禁用 802.11ac
    bl_mod_params.ht_on = true;
    bl_mod_params.vht_on = false;

    // 配置 MCS 映射：允许 MCS 0-7（64-QAM 最高）
    bl_mod_params.mcs_map = 0xFFF2;  // MCS 0-7 支持，MCS 8-9 不支持

    // 启用 Short GI
    bl_mod_params.sgi = true;

    // 禁用 40MHz（使用 20MHz 信道以提高兼容性）
    bl_mod_params.use_2040 = false;

    // 配置监听间隔：每 3 个 Beacon 周期唤醒一次
    bl_mod_params.listen_itv = 3;

    // 启用电源管理
    bl_mod_params.ps_on = true;

    // 配置 U-APSD：使能语音和视频队列
    bl_mod_params.uapsd_queues = 0x03;
    bl_mod_params.uapsd_timeout = 100;
}

// 连接到 Wi-Fi 4 AP
int wifi4_sta_connect(const char *ssid, const char *psk)
{
    wifi4_connect_init();

    // 添加 Station 接口
    struct mm_add_if_req add_if_req = {
        .type = MM_STA,
        .p2p = false,
    };
    // mac_addr 需根据实际 MAC 地址填充
    // 发送 MM_ADD_IF_REQ 消息至 LMAC

    // 配置信道（以 2.4GHz 信道 6 为例）
    struct mm_set_channel_req chan_req = {
        .band = 0,              // 2.4GHz
        .type = 20,             // 20MHz 带宽
        .prim20_freq = 2437,    // 2437 MHz
        .center1_freq = 2437,
        .tx_power = 18,         // 18 dBm
    };
    // 发送 MM_SET_CHANNEL_REQ 消息

    // 发起扫描
    struct scan_start_req scan_req = {
        .vif_idx = 0,
        .chan_cnt = 1,
        // 填充信道信息
    };
    // 发送 SCAN_START_REQ 消息

    // 扫描结果返回后，调用连接流程
    // 连接流程涉及 802.11 认证与关联帧交换

    return 0;
}
```

### 40MHz 带宽配置示例

以下代码演示如何将 Wi-Fi 4 配置为 40MHz 带宽模式：

```c
void wifi4_enable_40MHz(void)
{
    // 启用 40MHz 信道带宽
    bl_mod_params.use_2040 = true;

    // 配置主信道和中心频率
    // 以 2.4GHz 信道 6（主）+ 信道 10（辅）为例
    struct mm_set_channel_req chan_req = {
        .band = 0,                // 2.4GHz 频段
        .type = 40,               // 40MHz 带宽
        .prim20_freq = 2437,      // 主 20MHz 信道：2437 MHz (CH 6)
        .center1_freq = 2447,     // 中心频率：2447 MHz
        .center2_freq = 0,        // 80+80 模式使用
        .tx_power = 18,           // 发射功率
    };

    // 发送 MM_SET_CHANNEL_REQ 至 LMAC
    // 固件将协商 HT 40MHz 操作
}

void wifi4_disable_40MHz(void)
{
    // 禁用 40MHz，回退至 20MHz
    bl_mod_params.use_2040 = false;

    // 重新配置信道为 20MHz
    struct mm_set_channel_req chan_req = {
        .band = 0,
        .type = 20,
        .prim20_freq = 2437,
        .center1_freq = 2437,
        .tx_power = 18,
    };

    // 发送 MM_SET_CHANNEL_REQ
}
```

### 电源管理配置示例

```c
void wifi4_power_save_config(void)
{
    // 全局电源管理使能
    bl_mod_params.ps_on = true;

    // 配置监听间隔：每 10 个 Beacon 周期唤醒一次
    bl_mod_params.listen_itv = 10;

    // 忽略广播/多播（节省接收功耗）
    bl_mod_params.listen_bcmc = false;

    // 配置 U-APSD
    bl_mod_params.uapsd_queues = 0x0F;  // 所有队列使能 U-APSD
    bl_mod_params.uapsd_timeout = 200;  // 200ms 超时

    // 设置 LP 时钟精度（影响 DTIM 计算）
    bl_mod_params.lp_clk_ppm = 100;  // 100 ppm

    // 发送 MM_SET_PS_MODE_REQ，启用动态电源管理
    struct mm_set_ps_mode_req ps_req = {
        .new_state = MM_PS_MODE_ON_DYN,
    };
    // 发送消息至 LMAC
}
```

## 限制与注意事项

1. **40MHz 在 2.4GHz 频段**：2.4GHz 频段仅有 3 个非重叠信道（1、6、11），40MHz 带宽会占用相邻信道，实际部署中应确保信道无干扰。

2. **Short GI 可靠性**：Short GI 对多径敏感，在复杂室内环境中可能导致包错误率上升。若连接不稳定，应禁用 Short GI。

3. **MCS 协商**：实际使用的 MCS 索引由 AP 与 STA 双方能力集协商决定，Host 端配置仅为请求值。

4. **电源管理与延迟**：启用深度节能会引入额外唤醒延迟，对实时语音或游戏等低延迟应用不利。

5. **VHT 参数**：本芯片 `vht_on` 应始终保持 `false`，因为硬件不支持 802.11ac VHT。

## 参考

- IEEE Std 802.11n-2009: Amendment 5: Enhancements for Higher Throughput
- `lmac_msg.h`: LMAC 消息结构与枚举定义
- `bl_mod_params.h`: Wi-Fi 驱动参数配置结构
- `bl_platform.h`: 平台硬件抽象层定义
