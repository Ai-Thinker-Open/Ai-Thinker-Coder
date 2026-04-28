# lmac154 技术文档

## 概述

lmac154 是博流（Bouffalo）芯片系列中实现的 IEEE 802.15.4 MAC 层协议栈，为 Thread 和 Zigbee 等低速无线个人局域网（LR-WPAN）协议提供物理层和媒体访问控制层抽象。该模块工作在 2.4GHz ISM 频段，支持频道 11 至频道 26（共 16 个频道），符合 802.15.4-2015 标准。

lmac154 驱动版本为 1.7.4，提供完整的帧收发、射频状态管理、地址配置、CCA（Clear Channel Assessment）以及 AES-CCM 加密等核心功能。该模块是 Thread 协议的物理层和 MAC 层抽象基础，上层协议栈（如 OpenThread）通过 lmac154 提供的接口完成无线通信。

---

## 核心数据类型

### 频道类型 `lmac154_channel_t`

定义 802.15.4 工作频道，范围从频道 11 到频道 26：

```c
typedef enum {
    LMAC154_CHANNEL_NONE = -1,
    LMAC154_CHANNEL_11 = 0,
    LMAC154_CHANNEL_12,
    LMAC154_CHANNEL_13,
    LMAC154_CHANNEL_14,
    LMAC154_CHANNEL_15,
    LMAC154_CHANNEL_16,
    LMAC154_CHANNEL_17,
    LMAC154_CHANNEL_18,
    LMAC154_CHANNEL_19,
    LMAC154_CHANNEL_20,
    LMAC154_CHANNEL_21,
    LMAC154_CHANNEL_22,
    LMAC154_CHANNEL_23,
    LMAC154_CHANNEL_24,
    LMAC154_CHANNEL_25,
    LMAC154_CHANNEL_26,
} lmac154_channel_t;
```

### 发射功率类型 `lmac154_tx_power_t`

定义 8 级发射功率等级，从 0 dBm 到 7 dBm：

```c
typedef enum {
    LMAC154_TX_POWER_0dBm = 0,
    LMAC154_TX_POWER_1dBm = 1,
    LMAC154_TX_POWER_2dBm = 2,
    LMAC154_TX_POWER_3dBm = 3,
    LMAC154_TX_POWER_4dBm = 4,
    LMAC154_TX_POWER_5dBm = 5,
    LMAC154_TX_POWER_6dBm = 6,
    LMAC154_TX_POWER_7dBm = 7,
} lmac154_tx_power_t;
```

### 中断回调类型 `lmac154_isr_t`

用于注册硬件中断处理回调函数：

```c
typedef void (*lmac154_isr_t)(void);
```

### 数据速率类型 `lmac154_data_rate_t`

支持四种数据速率模式：

```c
typedef enum {
    LMAC154_DATA_RATE_250K = 0,  // 250 kbps（默认）
    LMAC154_DATA_RATE_500K = 1,  // 500 kbps
    LMAC154_DATA_RATE_1M   = 2,  // 1 Mbps
    LMAC154_DATA_RATE_2M   = 3,  // 2 Mbps
} lmac154_data_rate_t;
```

### CCA 模式类型 `lmac154_cca_mode_t`

定义信道空闲评估模式：

```c
typedef enum {
    LMAC154_CCA_MODE_ED        = 0,  // 能量检测
    LMAC154_CCA_MODE_CS        = 1,  // 载波侦听
    LMAC154_CCA_MODE_ED_AND_CS = 2,  // 能量检测与载波侦听（默认）
    LMAC154_CCA_MODE_ED_OR_CS  = 3,  // 能量检测或载波侦听
} lmac154_cca_mode_t;
```

### 帧类型 `lmac154_frame_type_t`

```c
typedef enum {
    LMAC154_FRAME_TYPE_BEACON = 0x01,  // 信标帧
    LMAC154_FRAME_TYPE_DATA   = 0x02,  // 数据帧
    LMAC154_FRAME_TYPE_ACK    = 0x04,  // 确认帧
    LMAC154_FRAME_TYPE_CMD    = 0x08,  // 命令帧
    LMAC154_FRAME_TYPE_MPP    = 0x10,  // 多用途帧
} lmac154_frame_type_t;
```

### 射频状态类型 `lmac154_rf_state_t`

```c
typedef enum {
    LMAC154_RF_STATE_RX_TRIG   = 1,  // RX 触发状态
    LMAC154_RF_STATE_RX        = 2,  // RX 运行状态
    LMAC154_RF_STATE_RX_DOING  = 3,  // RX 进行中
    LMAC154_RF_STATE_ACK_DOING = 4,  // ACK 发送中
    LMAC154_RF_STATE_TX        = 5,  // TX 运行状态
    LMAC154_RF_STATE_CSMA      = 6,  // CSMA/CA 运行中
    LMAC154_RF_STATE_IDLE      = 7,  // 空闲状态
} lmac154_rf_state_t;
```

### 发送状态类型 `lmac154_tx_status_t`

```c
typedef enum {
    LMAC154_TX_STATUS_TX_FINISHED = 0,  // 发送完成
    LMAC154_TX_STATUS_CSMA_FAILED = 1,  // CSMA 失败
    LMAC154_TX_STATUS_TX_ABORTED  = 2,  // 发送中止
    LMAC154_TX_STATUS_HW_ERROR    = 3,  // 硬件错误
    LMAC154_TX_STATUS_DELAY_ERROR = 4,  // 延迟错误
    LMAC154_TX_STATUS_NO_ACK      = 5,  // 未收到 ACK
    LMAC154_TX_STATUS_ACKED       = 6,  // 已确认
    LMAC154_TX_STATUS_CCA_FAILED  = 7,  // CCA 失败
    LMAC154_TX_STATUS_MAX         = 8,
} lmac154_tx_status_t;
```

### 接收状态类型 `lmac154_rx_status_t`

```c
typedef enum {
    LMAC154_RX_STATUS_NONE     = 0,  // 无事件
    LMAC154_RX_STATUS_RX_DONE  = 1,  // 接收完成
    LMAC154_RX_STATUS_ACK_SENT = 2,  // ACK 已发送
    LMAC154_RX_STATUS_ACK_ERR  = 3,  // ACK 错误
} lmac154_rx_status_t;
```

### 回调函数类型

发送完成回调：

```c
typedef void (*lmac154_txDoneCallback_t)(lmac154_tx_status_t status, 
                                          uint32_t *extra, uint32_t extra_len);
```

接收完成回调：

```c
typedef void (*lmac154_rxDoneCallback_t)(lmac154_rx_status_t status, 
                                         lmac154_receiveInfo_t *info, uint32_t *);
```

---

## 寄存器操作宏

lmac154 提供一组寄存器访问宏，用于直接读写硬件寄存器：

### 基本读写宏

```c
#define M154_RD_WORD(addr)      (*((volatile uint32_t *)(uintptr_t)(addr)))
#define M154_WR_WORD(addr, val) ((*(volatile uint32_t *)(uintptr_t)(addr)) = (val))
```

- `M154_RD_WORD(addr)`：从指定地址读取 32 位数据
- `M154_WR_WORD(addr, val)`：向指定地址写入 32 位数据

### 寄存器域操作宏

```c
#define M154_SET_REG_BIT(val, bitname)    ((val) | (1U << bitname##_POS))
#define M154_CLR_REG_BIT(val, bitname)    ((val) & bitname##_UMSK)
#define M154_GET_REG_BITS_VAL(val, bitname) (((val) & bitname##_MSK) >> bitname##_POS)
#define M154_SET_REG_BITS_VAL(val, bitname, bitval) (((val) & bitname##_UMSK) | \
                                    ((uint32_t)(bitval) << bitname##_POS))
#define M154_IS_REG_BIT_SET(val, bitname) (((val) & (1U << (bitname##_POS))) != 0)
```

- `M154_SET_REG_BIT`：设置特定位
- `M154_CLR_REG_BIT`：清除特定位
- `M154_GET_REG_BITS_VAL`：获取寄存器域值
- `M154_SET_REG_BITS_VAL`：设置寄存器域值
- `M154_IS_REG_BIT_SET`：检测特定位是否设置

---

## 802.15.4 帧结构

### 帧控制字段（Frame Control Field）

帧控制字段占 2 字节，包含帧类型、安全启用、地址模式等关键信息。lmac154_frame.h 中定义了丰富的帧解析宏：

```c
// 帧类型
#define LMAC154_FRAME_CONTROL_FRAME_TYPE_MASK    (7)
#define LMAC154_FRAME_CONTROL_FRAME_TYPE_BEACON (0)
#define LMAC154_FRAME_CONTROL_FRAME_TYPE_DATA   (1)
#define LMAC154_FRAME_CONTROL_FRAME_TYPE_ACK    (2)
#define LMAC154_FRAME_CONTROL_FRAME_TYPE_CMD   (3)
#define LMAC154_FRAME_CONTROL_FRAME_TYPE_MPP    (5)

// 安全与pending位
#define LMAC154_FRAME_SECURITY_MASK            (1 << 3)
#define LMAC154_FRAME_FRAME_PENDING_MASK       (1 << 4)
#define LMAC154_FRAME_ACK_REQUEST_MASK         (1 << 5)
#define LMAC154_FRAME_PANID_COMPRESSION       (1 << 6)

// 地址模式
#define LMAC154_FRAME_ADDR_DEST_NONE   (0 << 10)   // 无目标地址
#define LMAC154_FRAME_ADDR_DEST_SHORT  (2 << 10)   // 短地址
#define LMAC154_FRAME_ADDR_DEST_EXT    (3 << 10)    // 扩展地址

#define LMAC154_FRAME_ADDR_SRC_NONE    (0 << 14)
#define LMAC154_FRAME_ADDR_SRC_SHORT   (2 << 14)
#define LMAC154_FRAME_ADDR_SRC_EXT     (3 << 14)

// 帧版本
#define LMAC154_FRAME_VERSION_MASK     (3 << 12)
#define LMAC154_FRAME_VERSION_2006     (0 << 12)
#define LMAC154_FRAME_VERSION_2011     (1 << 12)
#define LMAC154_FRAME_VERSION_2015     (2 << 12)
```

### 帧结构常量

```c
#define LMAC154_LIFS                    40   // 长帧间间隔（符号数）
#define LMAC154_SIFS                    12   // 短帧间间隔（符号数）
#define LMAC154_PKT_MAX_LEN             127  // 最大 MPDU 长度
#define LMAC154_PREAMBLE_LEN             8    // 前导码长度
#define LMAC154_US_PER_SYMBOL            16   // 每符号微秒数
```

### MHR 解析函数

`lmac154_parse_mhr()` 是优化过的 32 位 MAC 头解析函数，用于快速提取帧中的地址信息：

```c
static inline uint32_t lmac154_parse_mhr(uint8_t *pr, uint8_t **a_dest_panid,
                                 uint8_t **a_dest_sa, uint8_t **a_dest_xa,
                                 uint8_t **a_src_sa, uint8_t **a_src_xa);
```

该函数一次性读取 32 位数据，解析帧控制字段、序列号、目标 PAN ID、目标地址、源 PAN ID 和源地址，返回 MAC 头长度。

---

## 核心 API 函数

### 初始化与复位

#### `lmac154_init()`

初始化 lmac154 硬件模块。在使用任何其他 API 之前必须先调用此函数。

```c
void lmac154_init(void);
```

#### `lmac154_reset()`

复位 lmac154 模块，将所有寄存器恢复至默认状态。

```c
void lmac154_reset(void);
```

#### `lmac154_resetTx()`

复位发送状态机，用于中止正在进行的发送操作。

```c
void lmac154_resetTx(void);
```

#### `lmac154_resetRx()`

复位接收状态机。

```c
void lmac154_resetRx(void);
```

---

### 频道与功率配置

#### `lmac154_set_channel()`

设置工作频道。默认值为 `LMAC154_CHANNEL_11`。

```c
void lmac154_setChannel(lmac154_channel_t ch_ind);
```

**参数：**
- `ch_ind`：频道索引，取值 `LMAC154_CHANNEL_11` 至 `LMAC154_CHANNEL_26`

**示例：**

```c
lmac154_setChannel(LMAC154_CHANNEL_15);
```

#### `lmac154_get_channel()`

获取当前工作频道。

```c
lmac154_channel_t lmac154_getChannel(void);
```

**返回值：**
- 当前频道索引

#### `lmac154_set_tx_power()`

设置发射功率。默认无默认值，使用前必须设置。

```c
void lmac154_setTxPower(lmac154_tx_power_t power_dbm);
```

**参数：**
- `power_dbm`：发射功率等级，取值 `LMAC154_TX_POWER_0dBm` 至 `LMAC154_TX_POWER_7dBm`

**示例：**

```c
lmac154_setTxPower(LMAC154_TX_POWER_5dBm);
```

#### `lmac154_get_tx_power()`

获取当前发射功率设置。

```c
lmac154_tx_power_t lmac154_getTxPower(void);
```

---

### 帧收发

#### `lmac154_send()`

发送 802.15.4 数据帧。这是最基本的发送接口。

```c
void lmac154_send(uint8_t *data, uint16_t length);
```

**参数：**
- `data`：指向数据缓冲区的指针
- `length`：数据长度（字节）

**备注：**
该函数内部自动添加 MHR（MAC 头），实际发送的 MPDU 长度不能超过 127 字节。

#### `lmac154_trigger_tx()`

触发发送，可选择是否使用 CSMA/CA。

```c
void lmac154_triggerTx(uint8_t *DataPtr, uint8_t length, uint8_t csma);
```

**参数：**
- `DataPtr`：数据缓冲区指针
- `length`：数据长度
- `csma`：0 表示不使用 CSMA/CA，1 表示使用 CSMA/CA

#### `lmac154_register_cb()`

注册接收中断回调函数。每当收到完整帧时，该回调会被调用。

```c
void lmac154_register_cb(lmac154_rxDoneCallback_t callback);
```

**参数：**
- `callback`：接收完成回调函数指针

#### `lmac154_register_event_callback()`

注册指定协议栈的接收事件回调，支持双协议栈模式。

```c
bool lmac154_registerEventCallback(lmac154_stack_idx_t stack_idx,
                                    lmac154_rxDoneCallback_t stack_rxDoneCallback);
```

**参数：**
- `stack_idx`：协议栈索引（`LMAC154_STACK_1` 或 `LMAC154_STACK_2`）
- `stack_rxDoneCallback`：接收完成回调

---

### 地址配置

#### PAN ID 配置

```c
void lmac154_setPanId(uint16_t pid);   // 设置 PAN ID
uint16_t lmac154_getPanId(void);       // 获取 PAN ID
```

#### 短地址配置

```c
void lmac154_setShortAddr(uint16_t sadr);  // 设置 16 位短地址
uint16_t lmac154_getShortAddr(void);       // 获取短地址
```

#### 扩展地址配置

```c
void lmac154_setLongAddr(uint8_t *ladr);   // 设置 64 位扩展地址
void lmac154_getLongAddr(uint8_t *ladr);   // 获取扩展地址
```

---

### 接收控制

#### 使能/禁用接收

```c
void lmac154_enableRx(void);    // 使能接收（默认禁用）
void lmac154_disableRx(void);   // 禁用接收
bool lmac154_isRxStateWhenIdle(void);
void lmac154_setRxStateWhenIdle(bool isRxOnWhenIdle);
```

#### 混杂模式

```c
void lmac154_enableRxPromiscuousMode(uint8_t enhanced_mode, uint8_t ignore_mpdu);
void lmac154_disableRxPromiscuousMode(void);
```

**参数说明：**
- `enhanced_mode`：0 标准模式，1 增强模式
- `ignore_mpdu`：0 从寄存器读取 MPDU，1 不读取

#### 帧类型过滤

```c
void lmac154_enableFrameTypeFiltering(uint8_t frame_types);
void lmac154_disableFrameTypeFiltering(void);
```

---

### CCA 与射频状态

#### 设置 CCA 模式

```c
void lmac154_setCCAMode(lmac154_cca_mode_t mode);
lmac154_cca_mode_t mode = LMAC154_CCA_MODE_ED_AND_CS;  // 默认
lmac154_setCCAMode(mode);
```

#### 设置 ED 阈值

```c
void lmac154_setEDThreshold(int threshold);  // 默认 -71 dBm
int threshold = -70;
lmac154_setEDThreshold(threshold);
```

#### 运行 CCA 检测

```c
uint8_t lmac154_runCCA(int *rssi);
```

**返回值：**
- 0：信道空闲
- 1：信道忙碌

#### 获取射频状态

```c
lmac154_rf_state_t lmac154_getRFState(void);
```

#### 获取 RSSI 和 LQI

```c
int lmac154_getRSSI(void);  // 获取 RSSI（dBm）
int lmac154_getLQI(void);   // 获取 LQI
```

---

### 其他配置 API

#### 数据速率

```c
void lmac154_setTxDataRate(lmac154_data_rate_t rate);  // 默认 250K
void lmac154_enableAutoRxDataRate(void);
void lmac154_disableAutoRxDataRate(void);
```

#### 重传次数

```c
void lmac154_setTxRetry(uint32_t num);  // 默认 0 次
```

#### 中断处理

```c
lmac154_isr_t lmac154_getInterruptHandler(void);
lmac154_isr_t lmac154_getInterruptCallback(void);
```

#### 版本信息

```c
uint32_t lmac154_getVersionNumber(void);
char *lmac154_getVersionString(void);
```

#### 国家码与功率限制

```c
bool lmac154_setCountryCode(const char *country_code);
void lmac154_setTxPowerWithPowerLimit(lmac154_tx_power_t power_dbm, 
                                       lmac154_channel_t ch_ind, 
                                       const char *country_code);
```

---

## 与 Thread 协议的关系

lmac154 是 Thread 协议的物理层和 MAC 层抽象。在 Thread 协议栈中，lmac154 负责：

- **物理层操作**：射频频道切换、发射功率控制、收发切换
- **MAC 层功能**：帧收发、CSMA/CA、ACK 处理、地址匹配
- **安全支撑**：AES-CCM 加密/解密支持，为 Thread 的 MAC 层安全提供基础

Thread 协议运行在 lmac154 之上，通过调用 lmac154 提供的发送和接收接口完成 MAC 帧的交互。lmac154 支持 802.15.4-2015 标准特性，包括增强型确认（Enhanced ACK）和信息元素（IE），这些都是 Thread 协议所需的特性。

---

## 代码示例

### 基本初始化与发送流程

```c
#include "lmac154.h"
#include "lmac154_frame.h"

// 接收回调函数
void my_rx_callback(lmac154_rx_status_t status, 
                    lmac154_receiveInfo_t *info, 
                    uint32_t *mpdu)
{
    if (status == LMAC154_RX_STATUS_RX_DONE) {
        uint8_t *data = (uint8_t *)mpdu;
        uint16_t len = info->rx_length;
        
        // 处理接收到的数据
        // ...
    }
}

void app_main(void)
{
    // 1. 初始化 lmac154
    lmac154_init();
    
    // 2. 设置 PAN ID 和短地址
    lmac154_setPanId(0x1234);
    lmac154_setShortAddr(0x0001);
    
    // 3. 设置工作频道
    lmac154_setChannel(LMAC154_CHANNEL_15);
    
    // 4. 设置发射功率
    lmac154_setTxPower(LMAC154_TX_POWER_5dBm);
    
    // 5. 注册接收回调
    lmac154_registerEventCallback(LMAC154_STACK_1, my_rx_callback);
    
    // 6. 使能接收
    lmac154_enableRx();
    
    // 7. 发送数据帧
    uint8_t tx_data[] = {
        0x12, 0x34,  // 目标 PAN ID
        0x00, 0x00,  // 目标地址（广播）
        0xFE, 0xCA,  // 源 PAN ID
        0x00, 0x01,  // 源地址
        0x01,        // 序列号
        0x02,        // 帧类型=数据, 协议版本=2006
        0x00,        // 安全关闭
        // ... 应用数据
    };
    
    lmac154_send(tx_data, sizeof(tx_data));
    
    while (1) {
        // 主循环
    }
}
```

### 高级发送（带参数）

```c
void advanced_send_example(void)
{
    // 配置发送参数
    lmac154_txParam_t tx_param = {0};
    
    uint8_t packet[] = { /* 帧数据 */ };
    
    tx_param.pkt = (uint32_t *)packet;
    tx_param.pkt_length = sizeof(packet);
    tx_param.tx_channel = LMAC154_CHANNEL_20 - LMAC154_CHANNEL_11;
    tx_param.is_cca = 1;                    // 启用 CCA
    tx_param.is_ack_required = 1;           // 请求 ACK
    tx_param.csma_ca_max_backoff = 4;       // 最大退避次数
    
    // 触发发送
    int ret = lmac154_triggerParamTx(&tx_param);
    if (ret != 0) {
        // 发送失败处理
    }
}
```

### CCA 信道检测

```c
void cca_check_example(void)
{
    int rssi;
    uint8_t result = lmac154_runCCA(&rssi);
    
    if (result == 0) {
        // 信道空闲，可以发送
        lmac154_send(data, len);
    } else {
        // 信道忙碌，等待后重试
    }
}
```

---

## 帧结构图示

### 802.15.4 MAC 帧格式

```
+--------+--------+------+----------+--------+--------+--------+------+
|        |        |      |          |        |        |        |      |
|  FCF   |  Seq   | Dst  |  Dst     |  Src   |  Src   |  Aux   |  Payload |
| (2B)   |  Num   | PAN  |  Addr    |  PAN   |  Addr  |  Sec   |  (nB) |
|        |  (1B)  | (2B) | (0/2/8B) |  (0/2B)|(0/2/8B)| Header |        |
|        |        |      |          |        |        | (5-14B)|        |
+--------+--------+------+----------+--------+--------+--------+------+
 |<---------- MAC Header (MHR) -------------------->|<---- MDS ---->|
```

- **FCF**：帧控制字段，2 字节
- **Seq Num**：序列号，1 字节
- **Dst PAN/Addr**：目标 PAN ID 和地址
- **Src PAN/Addr**：源 PAN ID 和地址
- **Aux Sec Header**：辅助安全头（当安全启用时）
- **Payload**：MAC 服务数据单元

---

## 参考

- IEEE 802.15.4-2015 Standard
- Bouffalo SDK lmac154 Component v1.7.4
- 源码文件：
  - `lmac154.h` - 主头文件
  - `lmac154_frame.h` - 帧结构定义
  - `lmac154_fpt.h` - 帧等待表相关定义
