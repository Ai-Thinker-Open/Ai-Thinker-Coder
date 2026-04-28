# CKS API Reference (BL616/BL618)

> **Source:** `bouffalo_sdk/drivers/lhal/include/bflb_cks.h`  
> **Implementation:** `bouffalo_sdk/drivers/lhal/src/bflb_cks.c`  
> **Register Header:** `bouffalo_sdk/drivers/lhal/include/hardware/cks_reg.h`

## Overview

CKS (Clock Security System / Checksum) 模块提供硬件校验和计算功能。它是一个简单的逐字节累加校验和引擎，支持可配置的大小端字节序（byte swap），适用于数据完整性快速校验。

**主要特性：**
- 硬件逐字节累加校验和
- 可配置大小端字节序（Little-Endian / Big-Endian）
- 16 位校验和输出
- 通过 ROM API 调用（支持 `romapi_` 快速路径）

## Base Address

| Peripheral | Base Address |
|------------|-------------|
| CKS | `0x2000A700` |

---

## 端序定义

```c
#define CKS_LITTLE_ENDIAN 0   // 小端模式
#define CKS_BIG_ENDIAN    1   // 大端模式
```

---

## LHAL API Functions

### bflb_cks_reset

复位校验和模块（清除内部累加器）。

```c
void bflb_cks_reset(struct bflb_device_s *dev);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | Device handle |

**说明:** 向 `CKS_CONFIG` 寄存器的 `CKS_CR_CKS_CLR` 位写 1 以清除累加器状态。

---

### bflb_cks_set_endian

设置校验和的字节序（大小端模式）。

```c
void bflb_cks_set_endian(struct bflb_device_s *dev, uint8_t endian);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | Device handle |
| `endian` | `uint8_t` | 字节序: `CKS_LITTLE_ENDIAN` (0) 或 `CKS_BIG_ENDIAN` (1) |

**说明:** 控制 `CKS_CR_CKS_BYTE_SWAP` 位 (bit 1)。

---

### bflb_cks_compute

对数据缓冲区执行硬件校验和计算。

```c
uint16_t bflb_cks_compute(struct bflb_device_s *dev, uint8_t *data, uint32_t length);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | Device handle |
| `data` | `uint8_t *` | 输入数据缓冲区指针 |
| `length` | `uint32_t` | 数据长度（字节数） |

**Returns:** `uint16_t` — 16 位校验和值

**说明:** 逐字节将数据写入 `CKS_DATA_IN` 寄存器，硬件自动累加，最后从 `CKS_OUT` 寄存器读取低 16 位结果。

---

## Usage Examples

### Example 1: 基本校验和计算

```c
#include "bflb_cks.h"

void cks_basic_example(void)
{
    struct bflb_device_s *cks;

    // 获取 CKS 设备句柄
    cks = bflb_device_get_by_name("cks");

    // 设置小端字节序
    bflb_cks_set_endian(cks, CKS_LITTLE_ENDIAN);

    // 待校验数据
    uint8_t data[] = {0x12, 0x34, 0x56, 0x78, 0x9A, 0xBC, 0xDE, 0xF0};

    // 计算校验和
    uint16_t checksum = bflb_cks_compute(cks, data, sizeof(data));
    printf("Checksum: 0x%04X\n", checksum);
}
```

### Example 2: 多次累加校验

```c
#include "bflb_cks.h"

void cks_accumulate_example(void)
{
    struct bflb_device_s *cks;
    cks = bflb_device_get_by_name("cks");

    bflb_cks_set_endian(cks, CKS_BIG_ENDIAN);

    // 复位累加器
    bflb_cks_reset(cks);

    // 分块累加
    uint8_t block1[] = {0x01, 0x02, 0x03, 0x04};
    uint8_t block2[] = {0x05, 0x06, 0x07, 0x08};

    bflb_cks_compute(cks, block1, sizeof(block1)); // 累加 block1
    bflb_cks_compute(cks, block2, sizeof(block2)); // 累加 block2

    // 注意: compute 返回的是累加后的结果，如需重新开始则先调用 reset
}
```

### Example 3: 固件完整性校验

```c
#include "bflb_cks.h"

bool verify_firmware_checksum(uint8_t *fw_data, uint32_t fw_len, uint16_t expected_cks)
{
    struct bflb_device_s *cks;
    cks = bflb_device_get_by_name("cks");

    bflb_cks_set_endian(cks, CKS_LITTLE_ENDIAN);

    uint16_t actual = bflb_cks_compute(cks, fw_data, fw_len);

    if (actual == expected_cks) {
        printf("Firmware checksum OK: 0x%04X\n", actual);
        return true;
    } else {
        printf("Checksum mismatch! Expected: 0x%04X, Got: 0x%04X\n",
               expected_cks, actual);
        return false;
    }
}
```

---

## Register-Level Reference

### CKS Register Offsets

| Offset | Register | Description |
|--------|----------|-------------|
| `0x00` | `CKS_CONFIG` | 控制寄存器 |
| `0x04` | `CKS_DATA_IN` | 数据输入寄存器 |
| `0x08` | `CKS_OUT` | 校验和输出寄存器 |

### Register Bitfields

#### CKS_CONFIG (0x00)

| Bit(s) | Field | Description |
|--------|-------|-------------|
| 0 | `CKS_CR_CKS_CLR` | 写 1 清除累加器 |
| 1 | `CKS_CR_CKS_BYTE_SWAP` | 字节序交换: 0=小端, 1=大端 |

```c
#define CKS_CR_CKS_CLR          (1<<0U)
#define CKS_CR_CKS_BYTE_SWAP    (1<<1U)
```

#### CKS_DATA_IN (0x04)

| Bit(s) | Field | Description |
|--------|-------|-------------|
| 7:0 | `CKS_DATA_IN` | 8 位数据输入（写操作触发累加） |

```c
#define CKS_DATA_IN_SHIFT       (0U)
#define CKS_DATA_IN_MASK        (0xff<<CKS_DATA_IN_SHIFT)
```

#### CKS_OUT (0x08)

| Bit(s) | Field | Description |
|--------|-------|-------------|
| 15:0 | `CKS_OUT` | 16 位校验和输出 |

```c
#define CKS_OUT_SHIFT           (0U)
#define CKS_OUT_MASK            (0xffff<<CKS_OUT_SHIFT)
```

### Direct Register Access Example

```c
#include "hardware/cks_reg.h"

// CKS 基地址: 0x2000A700
#define CKS_BASE_ADDR   0x2000A700

void cks_direct_example(uint8_t *data, uint32_t len)
{
    uint32_t cks_config = CKS_BASE_ADDR + CKS_CONFIG_OFFSET;   // 0x2000A700
    uint32_t cks_data_in = CKS_BASE_ADDR + CKS_DATA_IN_OFFSET; // 0x2000A704
    uint32_t cks_out     = CKS_BASE_ADDR + CKS_OUT_OFFSET;     // 0x2000A708

    // 清除累加器
    uint32_t regval = *(volatile uint32_t *)cks_config;
    regval |= CKS_CR_CKS_CLR;
    *(volatile uint32_t *)cks_config = regval;

    // 设置小端模式 (清除 bit 1)
    regval &= ~CKS_CR_CKS_BYTE_SWAP;
    *(volatile uint32_t *)cks_config = regval;

    // 逐字节输入数据
    for (uint32_t i = 0; i < len; i++) {
        *(volatile uint32_t *)cks_data_in = data[i];
    }

    // 读取校验和
    uint16_t result = (uint16_t)(*(volatile uint32_t *)cks_out & 0xFFFF);
    printf("Direct checksum: 0x%04X\n", result);
}
```
