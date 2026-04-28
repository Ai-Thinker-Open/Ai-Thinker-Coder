# IPC API Reference (BL616/BL618)

> **Source:** `bouffalo_sdk/drivers/lhal/include/bflb_ipc.h`  
> **Register Header:** `bouffalo_sdk/drivers/lhal/include/hardware/ipc_reg.h`  
> **Implementation:** `bouffalo_sdk/drivers/lhal/src/bflb_ipc.c`
>
> **⚠️ 芯片支持：** IPC (Inter-Processor Communication) 仅在 **BL618DG** 多核芯片上提供。BL616 单核芯片无此外设。

## Overview

IPC (Inter-Processor Communication / 核间通信) 模块为 BL618DG 多核系统中的 AP 核心（Application Processor）和 NP 核心（Network Processor）之间提供硬件级通信机制。每个 IPC 实例提供 32 个双向信号位，通过触发-应答-中断机制实现核间同步和消息传递。

**主要特性：**
- 2 个 IPC 实例 (IPC0, IPC1)
- 每实例 32 位通信通道
- 双向通信：AP → NP 和 NP → AP
- 硬件中断触发机制
- 可独立掩码每个位的中断

## Base Address

| 外设 | 基地址 | 说明 |
|------|--------|------|
| IPC0 | `0x20013000` | IPC 实例 0 |
| IPC1 | `0x20016000` | IPC 实例 1 |

> BL616 无此外设。

---

## 位掩码宏

用于操作 IPC 通道位：

```c
#define IPC_BITS_MAX   (32)            /* 最大位数 */
#define IPC_BITS_ALL   (0xffffffff)    /* 所有 32 位 */
#define IPC_BIT_NUM(n) ((0x01 << n) & IPC_BITS_ALL)  /* 特定位 n (0-31) */
```

**示例：**
```c
uint32_t channel_0 = IPC_BIT_NUM(0);   // 0x00000001
uint32_t channel_5 = IPC_BIT_NUM(5);   // 0x00000020
uint32_t multiple  = IPC_BIT_NUM(0) | IPC_BIT_NUM(3);  // 0x00000009
```

---

## LHAL API Functions

### bflb_ipc_init

初始化 IPC 实例。

```c
void bflb_ipc_init(struct bflb_device_s *dev);
```

**说明:** 清除所有 IPC 通道（ACK）并屏蔽所有中断。根据 `dev->sub_idx` 选择 AP→NP 或 NP→AP 方向：
- `sub_idx = 0`：AP 核心端（清除 AP2NP_ACK 和 AP2NP_UNMASK）
- `sub_idx = 1`：NP 核心端（清除 NP2AP_ACK 和 NP2AP_UNMASK）

---

### bflb_ipc_deinit

反初始化 IPC 实例。

```c
void bflb_ipc_deinit(struct bflb_device_s *dev);
```

---

### bflb_ipc_int_mask

屏蔽指定的 IPC 中断通道。

```c
void bflb_ipc_int_mask(struct bflb_device_s *dev, uint32_t ipc_bits);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | IPC 设备句柄 |
| `ipc_bits` | `uint32_t` | 要屏蔽的位掩码（使用 `IPC_BIT_NUM(n)` 或 `IPC_BITS_ALL`） |

---

### bflb_ipc_int_unmask

取消屏蔽（使能）指定的 IPC 中断通道。

```c
void bflb_ipc_int_unmask(struct bflb_device_s *dev, uint32_t ipc_bits);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | IPC 设备句柄 |
| `ipc_bits` | `uint32_t` | 要取消屏蔽的位掩码 |

---

### bflb_ipc_trig

触发 IPC 信号，向对端核心发送中断。

```c
void bflb_ipc_trig(struct bflb_device_s *dev, uint32_t ipc_bits);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | IPC 设备句柄 |
| `ipc_bits` | `uint32_t` | 要触发的位掩码 |

**说明:** 写入指定位的触发寄存器，硬件自动向对端核心产生中断（如果该位未被屏蔽）。

---

### bflb_ipc_clear

清除（应答）接收到的 IPC 信号。

```c
void bflb_ipc_clear(struct bflb_device_s *dev, uint32_t ipc_bits);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | IPC 设备句柄 |
| `ipc_bits` | `uint32_t` | 要清除的位掩码 |

**说明:** 写入 ACK 寄存器清除指定的 IPC 位。接收方处理完信号后应调用此函数清除状态。

---

### bflb_ipc_get_sta

获取 IPC 原始状态（RAW Status）。

```c
uint32_t bflb_ipc_get_sta(struct bflb_device_s *dev);
```

**Returns:** 32 位原始状态值（对端触发但尚未应答的位）

---

### bflb_ipc_get_intsta

获取 IPC 中断状态（Masked Status）。

```c
uint32_t bflb_ipc_get_intsta(struct bflb_device_s *dev);
```

**Returns:** 32 位中断状态值（已触发且未被屏蔽的位），即 `RAW_STATUS & ~MASK`

---

## 通信模型

### AP → NP 通信流程

```
AP Core (sender)                    NP Core (receiver)
     |                                     |
     |-- bflb_ipc_trig(bits) -->           |
     |   (写入 AP2NP_TRIGGER)              |
     |                                     |--- IPC 中断触发
     |                                     |-- bflb_ipc_get_intsta()
     |                                     |-- 处理信号
     |                                     |-- bflb_ipc_clear(bits)
     |                                     |   (写入 NP2AP_ACK)
```

### NP → AP 通信流程

```
NP Core (sender)                    AP Core (receiver)
     |                                     |
     |-- bflb_ipc_trig(bits) -->           |
     |   (写入 NP2AP_TRIGGER)              |
     |                                     |--- IPC 中断触发
     |                                     |-- bflb_ipc_get_intsta()
     |                                     |-- 处理信号
     |                                     |-- bflb_ipc_clear(bits)
     |                                     |   (写入 AP2NP_ACK)
```

---

## Usage Examples

### Example 1: AP 端初始化（接收方）

```c
#include "bflb_ipc.h"

void ap_ipc_init(void)
{
    struct bflb_device_s *ipc0;

    // 获取 IPC0 设备句柄 (AP 端: sub_idx=0)
    ipc0 = bflb_device_get_by_name("ipc0");

    // 初始化 IPC
    bflb_ipc_init(ipc0);

    // 取消屏蔽通道 0 和通道 1 的中断
    bflb_ipc_int_unmask(ipc0, IPC_BIT_NUM(0) | IPC_BIT_NUM(1));
}
```

### Example 2: NP 端发送信号给 AP

```c
#include "bflb_ipc.h"

void np_send_signal_to_ap(void)
{
    struct bflb_device_s *ipc0;

    // NP 端获取 IPC0 (sub_idx=1)
    ipc0 = bflb_device_get_by_name("ipc0_np");

    // 触发通道 0 给 AP
    bflb_ipc_trig(ipc0, IPC_BIT_NUM(0));
}
```

### Example 3: AP 端中断处理

```c
#include "bflb_ipc.h"

void ipc_interrupt_handler(void)
{
    struct bflb_device_s *ipc0;
    ipc0 = bflb_device_get_by_name("ipc0");

    // 获取中断状态
    uint32_t status = bflb_ipc_get_intsta(ipc0);

    // 处理每个通道
    if (status & IPC_BIT_NUM(0)) {
        // 处理通道 0 信号
        handle_channel_0_message();
        bflb_ipc_clear(ipc0, IPC_BIT_NUM(0));
    }

    if (status & IPC_BIT_NUM(1)) {
        // 处理通道 1 信号
        handle_channel_1_message();
        bflb_ipc_clear(ipc0, IPC_BIT_NUM(1));
    }

    // 轮询直到所有信号处理完毕
    while (bflb_ipc_get_intsta(ipc0)) {
        uint32_t remaining = bflb_ipc_get_intsta(ipc0);
        // 处理剩余信号...
        bflb_ipc_clear(ipc0, remaining);
    }
}
```

### Example 4: 核间同步示例

```c
#include "bflb_ipc.h"

// 信号定义
#define IPC_SIGNAL_DATA_READY   IPC_BIT_NUM(0)
#define IPC_SIGNAL_ACK          IPC_BIT_NUM(1)
#define IPC_SIGNAL_ERROR        IPC_BIT_NUM(2)

// AP 端
void ap_ipc_sync_example(void)
{
    struct bflb_device_s *ipc0;
    ipc0 = bflb_device_get_by_name("ipc0");

    bflb_ipc_init(ipc0);
    bflb_ipc_int_unmask(ipc0, IPC_SIGNAL_DATA_READY | IPC_SIGNAL_ERROR);

    // 等待 NP 的数据就绪信号
    // ... 在中断中处理 ...
}

// NP 端
void np_ipc_sync_example(void)
{
    struct bflb_device_s *ipc0;
    ipc0 = bflb_device_get_by_name("ipc0_np");

    bflb_ipc_init(ipc0);

    // 准备数据...
    prepare_data();

    // 通知 AP 数据就绪
    bflb_ipc_trig(ipc0, IPC_SIGNAL_DATA_READY);

    // 等待 AP 应答
    bflb_ipc_int_unmask(ipc0, IPC_SIGNAL_ACK);
    // ... 在中断中处理 ...
}
```

---

## Register-Level Reference

### IPC 寄存器布局

IPC 模块包含两组对称的寄存器：AP→NP 通道和 NP→AP 通道。

#### AP → NP 通道寄存器

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `IPC_AP2NP_TRIGGER` | `0x00` | AP→NP 触发寄存器 (写入触发信号) |
| `IPC_NP2AP_RAW_STATUS` | `0x04` | NP→AP 原始状态 (AP 可读) |
| `IPC_NP2AP_ACK` | `0x08` | NP→AP 应答清除 (AP 写入清除) |
| `IPC_NP2AP_UNMASK_SET` | `0x0C` | NP→AP 中断使能 (写 1 取消屏蔽) |
| `IPC_NP2AP_UNMASK_CLEAR` | `0x10` | NP→AP 中断禁用 (写 1 屏蔽) |
| `IPC_NP2AP_LINE_SEL_LOW` | `0x14` | NP→AP 线路选择低 32 位 |
| `IPC_NP2AP_LINE_SEL_HIGH` | `0x18` | NP→AP 线路选择高 32 位 |
| `IPC_NP2AP_STATUS` | `0x1C` | NP→AP 中断状态 (已掩码) |

#### NP → AP 通道寄存器

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `IPC_NP2AP_TRIGGER` | `0x20` | NP→AP 触发寄存器 (NP 写入触发信号) |
| `IPC_AP2NP_RAW_STATUS` | `0x24` | AP→NP 原始状态 (NP 可读) |
| `IPC_AP2NP_ACK` | `0x28` | AP→NP 应答清除 (NP 写入清除) |
| `IPC_AP2NP_UNMASK_SET` | `0x2C` | AP→NP 中断使能 (写 1 取消屏蔽) |
| `IPC_AP2NP_UNMASK_CLEAR` | `0x30` | AP→NP 中断禁用 (写 1 屏蔽) |
| `IPC_AP2NP_LINE_SEL_LOW` | `0x34` | AP→NP 线路选择低 32 位 |
| `IPC_AP2NP_LINE_SEL_HIGH` | `0x38` | AP→NP 线路选择高 32 位 |
| `IPC_AP2NP_STATUS` | `0x3C` | AP→NP 中断状态 (已掩码) |

### 寄存器位域

所有 IPC 寄存器均为 32 位宽，每位对应一个 IPC 通道 (0–31)：

| Bit | 对应通道 |
|-----|---------|
| 0 | Channel 0 |
| 1 | Channel 1 |
| ... | ... |
| 31 | Channel 31 |

**Trigger 寄存器** (偏移 0x00 / 0x20)：写入 `1` 的位会触发对端中断（该位须未被屏蔽）。

**RAW_STATUS 寄存器** (偏移 0x04 / 0x24)：只读，显示对端触发且尚未被应答的所有位。

**ACK 寄存器** (偏移 0x08 / 0x28)：写入 `1` 清除对应的 RAW_STATUS 位。

**UNMASK_SET 寄存器** (偏移 0x0C / 0x2C)：写入 `1` 允许对应通道产生中断。

**UNMASK_CLEAR 寄存器** (偏移 0x10 / 0x30)：写入 `1` 屏蔽对应通道中断（不清除已触发状态）。

**STATUS 寄存器** (偏移 0x1C / 0x3C)：只读，返回 `RAW_STATUS & UNMASK`（已触发且未被屏蔽的位）。

### Direct Register Access Example

```c
#include "hardware/ipc_reg.h"
#include "bl618dg_memorymap.h"

void ipc_direct_send_signal(uint32_t bits)
{
    // 直接写入 AP→NP 触发寄存器
    putreg32(bits, IPC0_BASE + IPC_AP2NP_TRIGGER_OFFSET);
}

uint32_t ipc_direct_read_status(void)
{
    // 读取 NP→AP 中断状态
    return getreg32(IPC0_BASE + IPC_NP2AP_STATUS_OFFSET);
}

void ipc_direct_clear(uint32_t bits)
{
    // 清除 NP→AP 中断
    putreg32(bits, IPC0_BASE + IPC_NP2AP_ACK_OFFSET);
}
```

---

## 设备表配置

BL618DG 设备表中 IPC 的配置：

| 设备名 | 基地址 | 中断号 | sub_idx | 方向 |
|--------|--------|--------|---------|------|
| `ipc0` | `IPC0_BASE` (0x20013000) | `BL618DG_IRQ_IPC0_CH0` | 0 | AP 端 |
| `ipc0_np` | `IPC0_BASE` (0x20013000) | `BL618DG_IRQ_IPC0_CH0` | 1 | NP 端 |
| `ipc1` | `IPC1_BASE` (0x20016000) | `BL618DG_IRQ_IPC1_CH0` | 0 | AP 端 |
| `ipc1_np` | `IPC1_BASE` (0x20016000) | `BL618DG_IRQ_IPC1_CH0` | 1 | NP 端 |

获取设备句柄：
```c
struct bflb_device_s *ipc0_ap = bflb_device_get_by_name("ipc0");     // AP 端
struct bflb_device_s *ipc0_np = bflb_device_get_by_name("ipc0_np");  // NP 端
```

---

## 注意事项

1. **仅 BL618DG 可用：** IPC 外设专为多核 BL618DG 设计，BL616 单核芯片不含此模块。

2. **方向区分：** 同一 IPC 实例的 AP 端和 NP 端通过 `sub_idx` 字段区分（0 = AP, 1 = NP），二者操作不同的寄存器组。

3. **中断处理要求：** 接收端在 ISR 中处理完信号后必须调用 `bflb_ipc_clear()` 清除对应位，否则中断持续触发。

4. **ROM API 加速：** 大部分 IPC 函数在 ROM 中有对应的 `romapi_bflb_ipc_*` 实现，SDK 通过条件编译优先使用 ROM 版本以获得更快的执行速度。

5. **多通道并发：** 32 个通道可同时使用，通过 `IPC_BIT_NUM(n)` 组合实现多通道并行通信。
