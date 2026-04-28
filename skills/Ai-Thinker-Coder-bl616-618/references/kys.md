# KYS API Reference (BL616/BL618)

> **Source:** `bouffalo_sdk/drivers/lhal/include/bflb_kys.h`  
> **Register Header:** `bouffalo_sdk/drivers/lhal/include/hardware/kys_reg.h`  
> **Implementation:** `bouffalo_sdk/drivers/lhal/src/bflb_kys.c`
>
> **⚠️ 芯片支持：** KYS (Key Scan) 外设在 BL616 上不可用，仅在 **BL618DG** 多核芯片上提供。

## Overview

KYS (Key Scan / 矩阵键盘扫描) 模块提供硬件矩阵键盘扫描功能。它自动逐列驱动扫描，读取行输入来检测按键按下。支持去抖动（deglitch）和鬼键检测（ghost key detection），以及 FIFO 缓冲按键事件。

最大支持 8 列 × 8 行矩阵键盘。

## Base Address

| 芯片 | 外设 | 基地址 |
|------|------|--------|
| BL618DG | KYS | `0x2000F800` |

> BL616 无此外设。

---

## 配置结构体

### bflb_kys_config_s

```c
struct bflb_kys_config_s {
    uint8_t col;           /* 键盘列数，最大 8 */
    uint8_t row;           /* 键盘行数，最大 8 */
    uint8_t deglitch_en;   /* 去抖动使能 */
    uint8_t deglitch_cnt;  /* 去抖动计数 */
    uint8_t idle_duration; /* 列扫描间隔空闲时间 */
    uint8_t ghost_en;      /* 鬼键检测使能 */
};
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `col` | `uint8_t` | 矩阵键盘列数 (1–8) |
| `row` | `uint8_t` | 矩阵键盘行数 (1–8) |
| `deglitch_en` | `uint8_t` | 去抖动使能 (0=禁用, 1=使能) |
| `deglitch_cnt` | `uint8_t` | 去抖动计数 (0–15, 防抖阈值) |
| `idle_duration` | `uint8_t` | 列扫描空闲间隔 (0–3，时间单位) |
| `ghost_en` | `uint8_t` | 鬼键检测使能 (0=禁用, 1=使能) |

---

## 中断标志宏

### 中断使能标志 (KEYSCAN_INT_EN_*)

用于 `bflb_kys_int_enable()` 的 `flag` 参数：

| 宏 | 位 | 说明 |
|----|-----|------|
| `KEYSCAN_INT_EN_DONE` | 7 | 扫描完成中断 |
| `KEYSCAN_INT_EN_FIFOFULL` | 8 | FIFO 满中断 |
| `KEYSCAN_INT_EN_FIFOHALF` | 9 | FIFO 半满中断 |
| `KEYSCAN_INT_EN_FIFOQUARTER` | 10 | FIFO 四分之一满中断 |
| `KEYSCAN_INT_EN_FIFO_NONEMPTY` | 11 | FIFO 非空中断 |
| `KEYSCAN_INT_EN_GHOST` | 12 | 鬼键检测中断 |

> BL702L 芯片的中断清除标志：
> - `KEYSCAN_INT_CLR_DONE` — 清除扫描完成状态
> - `KEYSCAN_INT_CLR_FIFO` — 清除 FIFO 状态
> - `KEYSCAN_INT_CLR_GHOST` — 清除鬼键状态

---

## LHAL API Functions

### bflb_kys_init

初始化矩阵键盘扫描控制器。

```c
void bflb_kys_init(struct bflb_device_s *dev, const struct bflb_kys_config_s *config);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | KYS 设备句柄 |
| `config` | `const struct bflb_kys_config_s *` | 键盘扫描配置参数 |

---

### bflb_kys_enable

使能键盘扫描。

```c
void bflb_kys_enable(struct bflb_device_s *dev);
```

---

### bflb_kys_disable

禁用键盘扫描。

```c
void bflb_kys_disable(struct bflb_device_s *dev);
```

---

### bflb_kys_int_enable

使能或禁用键盘扫描中断。

```c
void bflb_kys_int_enable(struct bflb_device_s *dev, uint32_t flag, bool enable);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `flag` | `uint32_t` | 中断标志 (使用 `KEYSCAN_INT_EN_*` 宏，可 OR 组合) |
| `enable` | `bool` | `true` = 使能中断, `false` = 禁用中断 |

---

### bflb_kys_int_clear

清除中断标志。

```c
void bflb_kys_int_clear(struct bflb_device_s *dev, uint32_t flag);
```

---

### bflb_kys_get_int_status

获取当前中断状态（仅返回已使能的中断位）。

```c
uint32_t bflb_kys_get_int_status(struct bflb_device_s *dev);
```

**Returns:** 中断状态位掩码

---

### bflb_kys_get_fifo_info

获取 FIFO 缓冲区信息（仅 BL702L 支持）。

```c
void bflb_kys_get_fifo_info(struct bflb_device_s *dev, uint8_t *fifo_head, uint8_t *fifo_tail, uint8_t *fifo_valid_cnt);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `fifo_head` | `uint8_t *` | 输出: FIFO 头指针 |
| `fifo_tail` | `uint8_t *` | 输出: FIFO 尾指针 |
| `fifo_valid_cnt` | `uint8_t *` | 输出: FIFO 有效数据计数 |

---

### bflb_kys_read_keyvalue

读取按键值。

```c
uint8_t bflb_kys_read_keyvalue(struct bflb_device_s *dev, uint8_t index);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `index` | `uint8_t` | 按键索引 (BL702: 0–3, BL702L: 忽略) |

**Returns:** 按键码值 (uint8_t)

---

## Usage Examples

### Example 1: 基本键盘扫描初始化

```c
#include "bflb_kys.h"

void keyscan_init(void)
{
    struct bflb_device_s *kys;

    // 获取 KYS 设备句柄
    kys = bflb_device_get_by_name("kys");

    // 配置 4x4 矩阵键盘
    struct bflb_kys_config_s cfg = {
        .col = 4,            // 4 列
        .row = 4,            // 4 行
        .deglitch_en = 1,    // 使能去抖动
        .deglitch_cnt = 5,   // 去抖动计数 5
        .idle_duration = 1,  // 扫描间隔
        .ghost_en = 1,       // 使能鬼键检测
    };

    bflb_kys_init(kys, &cfg);
    bflb_kys_enable(kys);
}
```

### Example 2: 中断方式扫描键盘

```c
#include "bflb_kys.h"

void keyscan_interrupt_init(void)
{
    struct bflb_device_s *kys;
    kys = bflb_device_get_by_name("kys");

    // 初始化键盘
    struct bflb_kys_config_s cfg = {
        .col = 3, .row = 3,
        .deglitch_en = 1, .deglitch_cnt = 3,
        .idle_duration = 0, .ghost_en = 1,
    };
    bflb_kys_init(kys, &cfg);

    // 使能 FIFO 非空中断和扫描完成中断
    bflb_kys_int_enable(kys, KEYSCAN_INT_EN_FIFO_NONEMPTY | KEYSCAN_INT_EN_DONE, true);

    // 使能扫描
    bflb_kys_enable(kys);

    // 在中断回调中处理按键
    bflb_irq_attach(KYS_IRQ_NUM, kys_irq_handler, NULL);
    bflb_irq_enable(KYS_IRQ_NUM);
}

void kys_irq_handler(int irq, void *arg)
{
    struct bflb_device_s *kys;
    kys = bflb_device_get_by_name("kys");

    uint32_t int_status = bflb_kys_get_int_status(kys);

    if (int_status & KEYSCAN_INT_EN_FIFO_NONEMPTY) {
        // 读取按键值
        uint8_t key = bflb_kys_read_keyvalue(kys, 0);
        // 处理按键值...
        bflb_kys_int_clear(kys, KEYSCAN_INT_CLR_FIFO);
    }

    if (int_status & KEYSCAN_INT_EN_DONE) {
        bflb_kys_int_clear(kys, KEYSCAN_INT_CLR_DONE);
    }
}
```

---

## Register-Level Reference

### KYS 寄存器偏移

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `KYS_KS_CTRL` | `0x00` | 控制寄存器 |
| `KYS_KS_INT_EN` | `0x10` | 中断使能寄存器 |
| `KYS_KS_INT_STS` | `0x14` | 中断状态寄存器 |
| `KYS_KEYCODE_CLR` | `0x18` | 按键码/中断清除寄存器 |
| `KYS_KEYFIFO_IDX` | `0x30` | FIFO 索引寄存器 (BL702L+) |
| `KYS_KEYFIFO_VALUE` | `0x34` | FIFO 值寄存器 |

### KS_CTRL 控制寄存器 (0x00)

| Bits | Field | Description |
|------|-------|-------------|
| 0 | `KYS_KS_EN` | 键盘扫描使能 |
| 1 | `KYS_FIFO_MODE` | FIFO 模式使能 (BL702L+) |
| 2 | `KYS_GHOST_EN` | 鬼键检测使能 |
| 3 | `KYS_DEG_EN` | 去抖动使能 |
| 7–4 | `KYS_DEG_CNT` | 去抖动计数 (0–15) |
| 9–8 | `KYS_RC_EXT` | 行/列扩展 (空闲间隔: 0–3) |
| 18–16 | `KYS_ROW_NUM` | 行数 (0–7, 对应 1–8 行) |
| 23–20 | `KYS_COL_NUM` | 列数 (BL702L: 0–31, BL702: 0–7) |

### KS_INT_EN 中断使能寄存器 (0x10)

| Bits | Field | Description |
|------|-------|-------------|
| 7 | `KYS_DONE_INT_EN` | 扫描完成中断使能 |
| 8 | `KEYFIFO_FULL_INT_EN` | FIFO 满中断使能 |
| 9 | `KEYFIFO_HALF_INT_EN` | FIFO 半满中断使能 |
| 10 | `KEYFIFO_QUARTER_INT_EN` | FIFO 1/4 满中断使能 |
| 11 | `KEYFIFO_NONEMPTY_INT_EN` | FIFO 非空中断使能 |
| 12 | `GHOST_INT_EN` | 鬼键中断使能 |

### KS_INT_STS 中断状态寄存器 (0x14)

| Bits | Field | Description |
|------|-------|-------------|
| 7 | `KEYCODE_DONE` | 扫描完成 |
| 8 | `KEYFIFO_FULL` | FIFO 满 |
| 9 | `KEYFIFO_HALF` | FIFO 半满 |
| 10 | `KEYFIFO_QUARTER` | FIFO 1/4 满 |
| 11 | `KEYFIFO_NONEMPTY` | FIFO 非空 |
| 12 | `GHOST_DET` | 鬼键检测到 |

### KEYFIFO_IDX 索引寄存器 (0x30, BL702L+)

| Bits | Field | Description |
|------|-------|-------------|
| 2–0 | `FIFO_HEAD` | FIFO 头指针 |
| 10–8 | `FIFO_TAIL` | FIFO 尾指针 |
| 19–16 | `FIFO_VALID_CNT` | FIFO 有效数据计数 |

### Direct Register Access Example

```c
#include "hardware/kys_reg.h"
#include "bl618dg_memorymap.h"

void kys_direct_enable(void)
{
    uint32_t reg_val;

    // 读取控制寄存器
    reg_val = getreg32(KYS_BASE + KYS_KS_CTRL_OFFSET);

    // 使能键盘扫描
    reg_val |= KYS_KS_EN_MASK;

    putreg32(reg_val, KYS_BASE + KYS_KS_CTRL_OFFSET);
}

uint32_t kys_direct_get_int_status(void)
{
    uint32_t int_sts = getreg32(KYS_BASE + KYS_KS_INT_STS_OFFSET);
    uint32_t int_en = getreg32(KYS_BASE + KYS_KS_INT_EN_OFFSET);
    return (int_sts & int_en);
}
```
