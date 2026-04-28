# EF_CTRL (eFuse Controller) API Reference (BL616/BL618)

> **Source:** `bouffalo_sdk/drivers/lhal/include/bflb_ef_ctrl.h`  
> **Implementation:** `bouffalo_sdk/drivers/lhal/src/bflb_ef_ctrl.c`  
> **Register Header:** `bouffalo_sdk/drivers/soc/bl616/std/include/hardware/ef_ctrl_reg.h`  
> **Chip Support:** BL602, BL702/BL702L, BL616/BL616CL, BL618DG

## Overview

EF_CTRL (eFuse Controller) 模块提供对芯片 eFuse（一次性可编程存储器）的读写控制接口。eFuse 用于存储芯片出厂校准数据、安全密钥、功能配置位等不可变信息。控制器提供自动模式和手动模式两种操作方式，支持 direct（直接）读写和 common trim（通用校准参数）两种数据访问抽象。

**主要特性：**
- Direct 模式：基于 offset 的原始 eFuse 数据读写
- Common Trim 模式：基于名称的校准参数读写（如 ADC trim、电压 trim 等）
- 自动加载检测（Auto-Load Done）
- eFuse 编程（烧写）支持
- 多 Region 支持（部分芯片有 Region 0 和 Region 1）
- 时序参数可配置
- 忙碌状态检测
- 工具函数：parity 计算、bit 检测、zero count

---

## Base Address

| 芯片 | EF_CTRL 基地址 | eFuse Region 0 大小 | eFuse Region 1 大小 |
|------|---------------|-------------------|-------------------|
| BL616 | `0x20056000` | 512 bytes (16×32-bit) | — |
| BL618DG (A0) | `0x2000C000` | 256 bytes (8×32-bit) | — |
| BL618DG (非 A0) | `0x2000C000` | 256 bytes | 256 bytes |
| BL616CL | `0x20056000` | 128 bytes (4×32-bit) | — |
| BL702/BL702L | `0x40007000` | 128 bytes | — |
| BL602 | `0x40007000` | 128 bytes | — |

---

## 数据类型

### bflb_ef_ctrl_com_trim_cfg_t

eFuse Common Trim 配置描述符（用于 trim 列表）。

```c
typedef struct {
    char *name;           /* trim 名称 */
    uint16_t en_addr;     /* enable 位地址 (bit offset) */
    uint16_t parity_addr; /* parity 位地址 (bit offset) */
    uint16_t value_addr;  /* value 位地址 (bit offset) */
    uint16_t value_len;   /* value 位长度 */
} bflb_ef_ctrl_com_trim_cfg_t;
```

---

### bflb_ef_ctrl_com_trim_t

eFuse Common Trim 数据（通过名称读取后返回）。

```c
typedef struct {
    uint8_t en;     /* Enable 状态 */
    uint8_t parity; /* Trim 校验位 */
    uint8_t empty;  /* Trim 是否为空 */
    uint8_t len;    /* Trim value 位长度 */
    uint32_t value; /* Trim 值 */
} bflb_ef_ctrl_com_trim_t;
```

---

### bflb_ef_ctrl_para_t

eFuse 控制器时序参数（用于读写时的时序调整）。

```c
typedef struct {
    uint16_t pd_1st;   /* 稳定时间 */
    uint16_t pd_cs_s;  /* CS 建立时间 (>500ns) */
    uint16_t cs;       /* CS 宽度 (>6.6ns) */
    uint16_t rd_adr;   /* 地址读取时间 (>6.3ns) */
    uint16_t rd_dat;   /* 数据读取时间 (>199ns) */
    uint16_t rd_dmy;   /* 空周期 (>14.9ns) */
    uint16_t pd_cs_h;  /* CS 保持时间 (>1ns) */
    uint16_t ps_cs;    /* CS 间隔 (>50ns) */
    uint16_t wr_adr;   /* 地址写入时间 (>6.3ns) */
    uint16_t pp;       /* 编程脉冲宽度 (>11-13us) */
    uint16_t pi;       /* 编程间隔 (>14.9ns) */
} bflb_ef_ctrl_para_t;
```

---

## AES 加密模式宏

仅 BL616CL / BL618DG:

```c
#define EF_CTRL_SF_AES_NONE  (0)  /* 无 AES 加密 */
#define EF_CTRL_SF_AES_128   (1)  /* AES-128 */
#define EF_CTRL_SF_AES_192   (2)  /* AES-192 */
#define EF_CTRL_SF_AES_256   (3)  /* AES-256 */
```

---

## API Functions

### 1. Trim 列表与自动加载

#### bflb_ef_ctrl_get_common_trim_list

获取 eFuse Common Trim 配置列表。

```c
uint32_t bflb_ef_ctrl_get_common_trim_list(const bflb_ef_ctrl_com_trim_cfg_t **trim_list);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `trim_list` | `const bflb_ef_ctrl_com_trim_cfg_t **` | 输出：trim 列表指针 |

**Returns:** trim 列表条目数量

**说明:** 返回芯片支持的 eFuse trim 参数列表（如 ADC offset、bandgap trim、LDO trim 等）。该列表用于 `bflb_ef_ctrl_read_common_trim()` 和 `bflb_ef_ctrl_write_common_trim()` 按名称索引。

---

#### bflb_ef_ctrl_autoload_done

检查 eFuse 自动加载是否完成。

```c
int bflb_ef_ctrl_autoload_done(struct bflb_device_s *dev);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | eFuse 设备句柄 |

**Returns:**

| 返回值 | 说明 |
|--------|------|
| `1` | 自动加载已完成 |
| `0` | 自动加载未完成 |

**说明:** 系统上电后 eFuse 控制器会自动将 eFuse 数据加载到 Shadow Register。此函数检查 `EF_CTRL_EF_IF_0_AUTOLOAD_DONE_MASK` 状态位。

---

### 2. 时序参数

#### bflb_ef_ctrl_set_para

设置 eFuse 控制器的读写时序参数。

```c
int bflb_ef_ctrl_set_para(bflb_ef_ctrl_para_t *para);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `para` | `bflb_ef_ctrl_para_t *` | 时序参数结构体指针 |

**Returns:** `0` 成功

**说明:** 将时序参数写入 `EF_IF_CYC_0` 和 `EF_IF_CYC_1` 寄存器，用于调整 eFuse 编程/读取的时序。BL616 使用默认参数，BL616CL 会根据系统时钟动态计算 `pp` 值。

---

### 3. Direct 读写

#### bflb_ef_ctrl_write_direct

直接写入 eFuse 数据（可选择性烧录）。

```c
void bflb_ef_ctrl_write_direct(struct bflb_device_s *dev, uint32_t offset, uint32_t *pword, uint32_t count, uint8_t program);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | eFuse 设备句柄 |
| `offset` | `uint32_t` | eFuse 写入偏移地址（字节） |
| `pword` | `uint32_t *` | 待写入数据缓冲区（word 对齐） |
| `count` | `uint32_t` | 写入的 word 数量 |
| `program` | `uint8_t` | `1` = 写入 Shadow Register 并烧录到 eFuse；`0` = 仅写 Shadow Register |

**说明:** 
- 在中断保护下执行
- 自动处理跨 Region 写入
- `program=1` 时会执行完整的编程流程（上电 → 编程 → 等待 busy → 断电）
- 自动调用 `bflb_ef_ctrl_update_para()` 更新时序参数
- 边界检查：超出范围时若 `program=1` 仅触发编程操作

---

#### bflb_ef_ctrl_read_direct

直接读取 eFuse 数据。

```c
void bflb_ef_ctrl_read_direct(struct bflb_device_s *dev, uint32_t offset, uint32_t *pword, uint32_t count, uint8_t reload);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | eFuse 设备句柄 |
| `offset` | `uint32_t` | eFuse 读取偏移地址（字节） |
| `pword` | `uint32_t *` | 读取数据缓冲区（word 对齐） |
| `count` | `uint32_t` | 读取的 word 数量 |
| `reload` | `uint8_t` | `1` = 从 eFuse 重新加载后再读；`0` = 从 Shadow Register 读取 |

**说明:**
- 在中断保护下执行
- 自动处理跨 Region 读取
- `reload=1` 时触发完整的 Load 流程并等待 auto-load done
- `reload=0` 时仅切换到 AHB Clock 后直接读取
- 自动调用 `bflb_ef_ctrl_update_para()` 更新时序参数

---

### 4. Common Trim 读写

#### bflb_ef_ctrl_read_common_trim

按名称读取 eFuse Common Trim 参数。

```c
void bflb_ef_ctrl_read_common_trim(struct bflb_device_s *dev, char *name, bflb_ef_ctrl_com_trim_t *trim, uint8_t reload);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | eFuse 设备句柄 |
| `name` | `char *` | Trim 参数名称（字符串匹配） |
| `trim` | `bflb_ef_ctrl_com_trim_t *` | 输出：读取到的 trim 数据 |
| `reload` | `uint8_t` | `1` = 重新加载；`0` = 直接读 Shadow Register |

**说明:**
- 通过 `bflb_ef_ctrl_get_common_trim_list()` 获取 trim 列表
- 按 `name` 字符串精确匹配查找对应的 trim 配置
- 自动解析跨 32-bit 边界的 value 字段（支持 64-bit 拼接）
- 读取 en、parity、value 位段，并判断 empty 状态
- 在中断保护下执行

---

#### bflb_ef_ctrl_write_common_trim

按名称写入 eFuse Common Trim 参数。

```c
void bflb_ef_ctrl_write_common_trim(struct bflb_device_s *dev, char *name, uint32_t value, uint8_t program);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | eFuse 设备句柄 |
| `name` | `char *` | Trim 参数名称 |
| `value` | `uint32_t` | 待写入的 trim 值 |
| `program` | `uint8_t` | `1` = 烧录到 eFuse（一次性）；`0` = 仅写 Shadow Register |

**说明:**
- 自动计算 parity 位并一同写入
- 设置 enable 位标记该 trim 有效
- 处理跨 32-bit 边界的 value 写入
- `program=1` 时执行 eFuse 编程流程（上电 → 编程 → 等待 → 断电）
- 在中断保护下执行

---

### 5. 状态与工具函数

#### bflb_ef_ctrl_busy

检查 eFuse Region 0 是否忙碌。

```c
int bflb_ef_ctrl_busy(struct bflb_device_s *dev);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | eFuse 设备句柄 |

**Returns:**

| 返回值 | 说明 |
|--------|------|
| `1` | eFuse 忙碌 |
| `0` | eFuse 空闲 |

**说明:** 检查 `EF_CTRL_EF_IF_0_BUSY_MASK` 状态位。编程操作完成后需等待 busy 清除。

---

#### bflb_ef_ctrl_busy_r1

检查 eFuse Region 1 是否忙碌。

```c
int bflb_ef_ctrl_busy_r1(struct bflb_device_s *dev);
```

> ⚠️ **条件:** 仅 `BL618DG && !CPU_MODEL_A0` 可用

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | eFuse 设备句柄 |

**Returns:**

| 返回值 | 说明 |
|--------|------|
| `1` | eFuse Region 1 忙碌 |
| `0` | eFuse Region 1 空闲 |

---

#### bflb_ef_ctrl_is_all_bits_zero

检查一个值中指定位段是否全为零。

```c
uint8_t bflb_ef_ctrl_is_all_bits_zero(uint32_t val, uint8_t start, uint8_t len);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `val` | `uint32_t` | 待检查的值 |
| `start` | `uint8_t` | 起始位位置 |
| `len` | `uint8_t` | 检查的位长度 |

**Returns:** `1` = 全零，`0` = 存在非零位

---

#### bflb_ef_ctrl_get_byte_zero_cnt

统计一个字节中零位的个数。

```c
uint32_t bflb_ef_ctrl_get_byte_zero_cnt(uint8_t val);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `val` | `uint8_t` | 待统计的字节 |

**Returns:** 零位个数（0-8）

---

#### bflb_ef_ctrl_get_trim_parity

计算 Trim 值的奇偶校验位。

```c
uint8_t bflb_ef_ctrl_get_trim_parity(uint32_t val, uint8_t len);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `val` | `uint32_t` | Trim 值 |
| `len` | `uint8_t` | 参与校验的位长度 |

**Returns:** 奇校验位（0 或 1）——1 的个数的 LSB

**说明:** 统计 `val` 的 `[0, len)` 位中 1 的个数，返回最低位。用于 eFuse trim 写入时自动填充 parity 位。

---

## 寄存器参考

### Region 0 控制寄存器 (BL616: offset 0x800 from base)

**`EF_CTRL_EF_IF_CTRL_0` (0x800)**

| Bits | Field | Description |
|------|-------|-------------|
| 0 | `AUTOLOAD_P1_DONE` | Phase 1 自动加载完成 |
| 1 | `AUTOLOAD_DONE` | 自动加载完成 |
| 2 | `BUSY` | eFuse 忙碌状态 |
| 3 | `RW` | 读写选择 (0=读, 1=写) |
| 4 | `TRIG` | 触发操作 |
| 5 | `MANUAL_EN` | 手动模式使能 (0=自动模式) |
| 6 | `CYC_MODIFY` | 时序参数修改使能 |
| 8-15 | `PROT_CODE_CTRL` | 控制寄存器保护码 (需写入 0xBF) |
| 16 | `POR_DIG` | 数字 POR 控制 |
| 17 | `PCLK_FORCE_ON` | 强制开启 PCLK |
| 18 | `AUTO_RD_EN` | 自动读取使能 |
| 19 | `CYC_MODIFY_LOCK` | 时序修改锁定 |
| 20 | `INT` | 中断状态 |
| 21 | `INT_CLR` | 中断清除 |
| 22 | `INT_SET` | 中断置位 |
| 24-31 | `PROT_CODE_CYC` | 时序寄存器保护码 (需写入 0xBF) |

### 时序配置寄存器

**`EF_CTRL_EF_IF_CYC_0` (0x804)**

| Bits | Field | Description |
|------|-------|-------------|
| 0-5 | `RD_DMY` | 读取空周期 |
| 6-11 | `RD_DAT` | 读取数据周期 |
| 12-17 | `RD_ADR` | 读取地址周期 |
| 18-23 | `CS` | CS 信号宽度 |
| 24-31 | `PD_CS_S` | CS 建立时间 |

**`EF_CTRL_EF_IF_CYC_1` (0x808)**

| Bits | Field | Description |
|------|-------|-------------|
| 0-5 | `PI` | 编程间隔 |
| 6-13 | `PP` | 编程脉冲宽度 |
| 14-19 | `WR_ADR` | 写入地址周期 |
| 20-25 | `PS_CS` | CS 间隔 |
| 26-31 | `PD_CS_H` | CS 保持时间 |

### eFuse 配置寄存器 (BL616)

**`EF_CTRL_EF_IF_CFG_0` (0x814)**

| Bits | Field | Description |
|------|-------|-------------|
| 0-1 | `SF_AES_MODE` | AES 加密模式 |
| 2 | `AI_DIS` | AI 功能禁用 |
| 3 | `CPU0_DIS` | CPU0 禁用 |
| 4-5 | `SBOOT_EN` | Secure Boot 使能 |
| 6-9 | `UART_DIS` | UART 禁用 |
| 10 | `BUS_RMP_SW_EN` | Bus Remap 软件使能 |
| 11 | `BUS_RMP_DIS` | Bus Remap 禁用 |
| 12-13 | `SF_KEY_RE_SEL` | Flash Key Region 选择 |
| 14 | `SDU_DIS` | SDU 禁用 |
| 15 | `BTDM_DIS` | BTDM 禁用 |
| 16 | `WIFI_DIS` | WiFi 禁用 |
| 17 | `KEY_ENC_EN` | Key 加密使能 |
| 18 | `CAM_DIS` | Camera 禁用 |
| 19 | `M154_DIS` | 802.15.4 禁用 |
| 20 | `CPU1_DIS` | CPU1 (NP) 禁用 |
| 21 | `CPU_RST_DBG_DIS` | CPU Reset Debug 禁用 |
| 22 | `SE_DBG_DIS` | SE Debug 禁用 |
| 23 | `EFUSE_DBG_DIS` | eFuse Debug 禁用 |
| 24-25 | `DBG_JTAG_1_DIS` | JTAG1 Debug 禁用 |
| 26-27 | `DBG_JTAG_0_DIS` | JTAG0 Debug 禁用 |
| 28-31 | `DBG_MODE` | Debug 模式 |

---

## Usage Examples

### Example 1: 检查 eFuse 自动加载状态

```c
#include "bflb_ef_ctrl.h"
#include "bflb_core.h"

void wait_efuse_autoload(void)
{
    struct bflb_device_s *ef_ctrl;
    
    ef_ctrl = bflb_device_get_by_name("ef_ctrl");
    
    // 轮询等待 eFuse 自动加载完成
    while (!bflb_ef_ctrl_autoload_done(ef_ctrl)) {
        bflb_mtimer_delay_ms(1);
    }
    
    printf("eFuse auto-load complete\n");
}
```

### Example 2: 读取 eFuse 原始数据

```c
#include "bflb_ef_ctrl.h"

void read_efuse_raw_data(void)
{
    struct bflb_device_s *ef_ctrl;
    uint32_t efuse_data[4];  // BL616 R0 = 512 bytes = 128 words
    
    ef_ctrl = bflb_device_get_by_name("ef_ctrl");
    
    // 读取前 4 words (offset 0, 16 bytes)
    bflb_ef_ctrl_read_direct(ef_ctrl, 0, efuse_data, 4, 1);
    
    printf("eFuse[0x00]: 0x%08lx\n", efuse_data[0]);
    printf("eFuse[0x04]: 0x%08lx\n", efuse_data[1]);
    printf("eFuse[0x08]: 0x%08lx\n", efuse_data[2]);
    printf("eFuse[0x0C]: 0x%08lx\n", efuse_data[3]);
}
```

### Example 3: 读取 Common Trim 参数

```c
#include "bflb_ef_ctrl.h"

void read_adc_trim(void)
{
    struct bflb_device_s *ef_ctrl;
    bflb_ef_ctrl_com_trim_t trim;
    
    ef_ctrl = bflb_device_get_by_name("ef_ctrl");
    
    // 按名称读取 ADC offset trim (实际名称取决于芯片 trim 列表)
    bflb_ef_ctrl_read_common_trim(ef_ctrl, "adc_offset", &trim, 1);
    
    if (!trim.empty) {
        printf("ADC Offset Trim:\n");
        printf("  Enabled: %d\n", trim.en);
        printf("  Parity:  %d\n", trim.parity);
        printf("  Value:   0x%lx (%lu)\n", trim.value, trim.value);
        printf("  Length:  %d bits\n", trim.len);
    } else {
        printf("ADC Offset Trim is empty (not programmed)\n");
    }
}
```

### Example 4: 遍历所有 Common Trim 参数

```c
#include "bflb_ef_ctrl.h"

void list_all_trims(void)
{
    struct bflb_device_s *ef_ctrl;
    const bflb_ef_ctrl_com_trim_cfg_t *trim_list;
    uint32_t trim_count;
    
    ef_ctrl = bflb_device_get_by_name("ef_ctrl");
    
    // 获取 trim 列表
    trim_count = bflb_ef_ctrl_get_common_trim_list(&trim_list);
    
    printf("Total %lu common trim parameters:\n", trim_count);
    
    for (uint32_t i = 0; i < trim_count; i++) {
        bflb_ef_ctrl_com_trim_t trim;
        
        bflb_ef_ctrl_read_common_trim(ef_ctrl, trim_list[i].name, &trim, 0);
        
        printf("  [%lu] %-20s en=%d parity=%d empty=%d len=%d value=0x%lx\n",
               i, trim_list[i].name, trim.en, trim.parity, 
               trim.empty, trim.len, trim.value);
    }
}
```

### Example 5: 写入并编程 eFuse（仅限开发/工厂使用）

```c
#include "bflb_ef_ctrl.h"

int program_efuse_example(void)
{
    struct bflb_device_s *ef_ctrl;
    uint32_t write_data[1] = { 0x12345678 };
    
    ef_ctrl = bflb_device_get_by_name("ef_ctrl");
    
    // ⚠️ 警告：eFuse 是一次性可编程存储器，编程后无法撤销！
    
    // 先检查 eFuse 是否忙碌
    if (bflb_ef_ctrl_busy(ef_ctrl)) {
        printf("eFuse is busy, cannot program now\n");
        return -1;
    }
    
    // 写入 Shadow Register 并烧录到 eFuse (offset=0x20, 1 word)
    bflb_ef_ctrl_write_direct(ef_ctrl, 0x20, write_data, 1, 1);
    
    // 等待编程完成
    uint32_t timeout = 100000;
    while (bflb_ef_ctrl_busy(ef_ctrl)) {
        timeout--;
        if (timeout == 0) {
            printf("eFuse program timeout!\n");
            return -1;
        }
        bflb_mtimer_delay_us(10);
    }
    
    // 验证写入
    uint32_t verify_data[1] = { 0 };
    bflb_ef_ctrl_read_direct(ef_ctrl, 0x20, verify_data, 1, 1);
    
    if (verify_data[0] == write_data[0]) {
        printf("eFuse program and verify OK\n");
        return 0;
    } else {
        printf("eFuse verify failed: wrote 0x%lx, read 0x%lx\n",
               write_data[0], verify_data[0]);
        return -1;
    }
}
```

### Example 6: 奇偶校验工具函数

```c
#include "bflb_ef_ctrl.h"

void parity_example(void)
{
    uint32_t value = 0xA5;  // 二进制: 1010 0101, 4个1
    
    // 计算奇校验位
    uint8_t parity = bflb_ef_ctrl_get_trim_parity(value, 8);
    printf("Value 0x%02lx: parity=%d\n", value, parity);
    // 输出: parity=0 (偶数个1)
    
    uint32_t value2 = 0x07;  // 二进制: 0000 0111, 3个1
    parity = bflb_ef_ctrl_get_trim_parity(value2, 8);
    printf("Value 0x%02lx: parity=%d\n", value2, parity);
    // 输出: parity=1 (奇数个1, LSB of count)
    
    // 检查特定位段是否全零
    uint8_t all_zero = bflb_ef_ctrl_is_all_bits_zero(0x00FF0000, 16, 8);
    printf("Bits [23:16] all zero: %d\n", all_zero);
    // 输出: 1 (true)
    
    // 统计零位个数
    uint32_t zero_cnt = bflb_ef_ctrl_get_byte_zero_cnt(0xF0);
    printf("0xF0 has %lu zero bits\n", zero_cnt);
    // 输出: 4
}
```

---

## 注意事项

1. **一次编程:** eFuse 是一次性可编程存储器，bit 只能从 0 编程为 1，不能从 1 恢复为 0。编程操作需谨慎。

2. **保护码:** 修改 eFuse 控制寄存器需要写入保护码 `0xBF` 到 `PROT_CODE_CTRL` 和 `PROT_CODE_CYC` 位段，防止误操作。

3. **中断保护:** 所有读写和编程函数都在 `bflb_irq_save()` / `bflb_irq_restore()` 保护下执行，确保操作原子性。

4. **Region 支持:** BL616 仅有 Region 0（512 bytes），BL618DG 非 A0 版本有 Region 0 和 Region 1。跨 Region 读写会自动处理。

5. **Shadow Register:** `program=0` 或 `reload=0` 时操作的是 Shadow Register（缓存副本），不会触发实际的 eFuse 硬件操作，速度快但断电丢失。上电后需等待 Auto-Load Done。

6. **TCM 段:** 所有 API 函数位于 TCM 段（`ATTR_TCM_SECTION`），确保执行期间不受 Flash 访问延迟影响。

7. **ROM API:** 函数内部优先调用 ROM API（`romapi_bflb_ef_ctrl_*`），减小代码体积。

| 操作模式 | 参数设置 | 适用场景 |
|---------|---------|---------|
| 只读不加载 | `reload=0` | 已知已完成自动加载，快速读取 |
| 读取+加载 | `reload=1` | 不确定状态，保证读最新 |
| 只写不编程 | `program=0` | 预写入，稍后统一编程 |
| 写入+编程 | `program=1` | 直接烧录到 eFuse（一次性） |
