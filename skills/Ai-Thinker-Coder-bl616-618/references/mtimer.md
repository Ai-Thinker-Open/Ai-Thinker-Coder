# MTIMER API Reference (BL616/BL618)

> **Source:** `bouffalo_sdk/drivers/lhal/include/bflb_mtimer.h`  
> **Implementation:** `bouffalo_sdk/drivers/lhal/src/bflb_mtimer.c`  
> **Chip Support:** BL602, BL702/BL702L, BL616/BL618, BL618DG

## Overview

MTIMER (Machine Timer) 是基于 RISC-V 架构的 Machine Timer 外设提供的 64 位高精度硬件定时器接口。它使用 RISC-V 核心的 `mtime`/`mtimecmp` 寄存器（BL602/BL702）或 CSI Core Timer（BL616/BL618），提供微秒级时间戳和精确延时功能。

**主要特性：**
- 64 位高精度时间戳（微秒级）
- 硬件定时器中断支持
- 微秒/毫秒延时函数（阻塞式）
- 默认频率 1 MHz（1 tick = 1 μs）
- 函数位于 TCM 段以提供低延迟访问

## MTIMER 频率

默认定时器频率为 **1 MHz**（每秒 1,000,000 ticks），可通过重写 `bflb_mtimer_get_freq()` 弱函数自定义。开启 `CONFIG_MTIMER_CUSTOM_FREQUENCE` 配置项后，时间计算会自动适配自定义频率。

---

## LHAL API Functions

### bflb_mtimer_config

配置 Machine Timer 中断。

```c
void bflb_mtimer_config(uint64_t ticks, void (*interruptfun)(void));
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `ticks` | `uint64_t` | 触发中断所需的 tick 数（默认 1 tick = 1 μs） |
| `interruptfun` | `void (*)(void)` | 中断回调函数指针 |

**说明:** 该函数配置定时器在 `ticks` 个 tick 后触发中断，使用 IRQ 7（Machine Timer 中断）。此后每个 `ticks` 周期重复触发。

---

### bflb_mtimer_get_freq

获取 Machine Timer 当前频率。

```c
uint32_t bflb_mtimer_get_freq(void);
```

**Returns:** 定时器频率（Hz），默认返回 `1000000` (1 MHz)

**说明:** 该函数为弱函数（`__WEAK`），可被用户重写以返回实际频率。

---

### bflb_mtimer_get_time_us

获取当前 Machine Timer 时间（微秒）。

```c
uint64_t bflb_mtimer_get_time_us(void);
```

**Returns:** 自启动以来的 64 位时间值（微秒），~584942 年才会溢出

**说明:** 函数位于 TCM 段，执行速度极快。内部采用双读防翻转机制（读取两次 low + high 确保一致性）。

---

### bflb_mtimer_get_time_ms

获取当前 Machine Timer 时间（毫秒）。

```c
uint64_t bflb_mtimer_get_time_ms(void);
```

**Returns:** 自启动以来的 64 位时间值（毫秒）

---

### bflb_mtimer_delay_us

微秒级阻塞延时。

```c
void bflb_mtimer_delay_us(uint32_t time);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `time` | `uint32_t` | 延时时长（微秒） |

**说明:** 忙等待方式。对于 BL616/BL618，支持 ROM API 加速。函数位于 TCM 段。

---

### bflb_mtimer_delay_ms

毫秒级阻塞延时。

```c
void bflb_mtimer_delay_ms(uint32_t time);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `time` | `uint32_t` | 延时时长（毫秒） |

---

### bflb_mtimer_set_val

直接设置 Machine Timer 计数值（仅 BL616CL 支持）。

```c
void bflb_mtimer_set_val(uint64_t val);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `val` | `uint64_t` | 要设置的定时器值 |

---

## Usage Examples

### Example 1: 基本延时

```c
#include "bflb_mtimer.h"

void delay_example(void)
{
    // 微秒延时
    bflb_mtimer_delay_us(100);   // 延时 100 μs

    // 毫秒延时
    bflb_mtimer_delay_ms(500);   // 延时 500 ms
}
```

### Example 2: 性能计时

```c
#include "bflb_mtimer.h"
#include "bflb_platform.h"

void benchmark_example(void)
{
    uint64_t start, end;

    start = bflb_mtimer_get_time_us();

    // 执行被测代码
    some_operation();

    end = bflb_mtimer_get_time_us();

    MSG("Operation took %llu us\r\n", (unsigned long long)(end - start));
}
```

### Example 3: 定时器中断

```c
#include "bflb_mtimer.h"

static volatile uint32_t tick_count = 0;

void my_timer_callback(void)
{
    tick_count++;
    // 每 1ms 调用一次
}

void timer_interrupt_example(void)
{
    // 配置 1ms 定时中断 (默认 1 MHz, 1000 ticks = 1 ms)
    bflb_mtimer_config(1000, my_timer_callback);

    while (1) {
        // 定时器在后台产生中断
        // tick_count 每秒增加 1000
    }
}
```

### Example 4: 超时判断

```c
#include "bflb_mtimer.h"

bool wait_for_condition_with_timeout(uint32_t timeout_ms)
{
    uint64_t start = bflb_mtimer_get_time_ms();

    while (!condition_met()) {
        if (bflb_mtimer_get_time_ms() - start >= timeout_ms) {
            return false;  // 超时
        }
    }
    return true;  // 条件满足
}
```

### Example 5: 微秒延时精度测试

```c
#include "bflb_mtimer.h"
#include "bflb_platform.h"

void delay_accuracy_test(void)
{
    uint64_t start, end;
    uint32_t delays[] = {1, 10, 50, 100, 500, 1000};

    for (int i = 0; i < sizeof(delays) / sizeof(delays[0]); i++) {
        start = bflb_mtimer_get_time_us();
        bflb_mtimer_delay_us(delays[i]);
        end = bflb_mtimer_get_time_us();

        MSG("Requested: %u us, Actual: %llu us\r\n",
            delays[i], (unsigned long long)(end - start));
    }
}
```

---

## 架构细节

### BL602 / BL702 / BL702L

这些芯片直接使用 RISC-V CLIC (Core Local Interrupt Controller) 中的 MTIME/MTIMECMP 寄存器：

- `CLIC_MTIME`: 64 位单调递增计数器
- `CLIC_MTIMECMP`: 64 位比较值寄存器

地址通过 `CLIC_CTRL_BASE + CLIC_MTIME_OFFSET` 和 `CLIC_CTRL_BASE + CLIC_MTIMECMP_OFFSET` 访问。

### BL616 / BL618 / BL618DG

这些芯片使用 CSI Core Timer (C-SKY 架构核心定时器)：

- `csi_coret_get_value()` / `csi_coret_get_valueh()`: 读取 64 位定时器值
- `csi_coret_config()`: 配置比较值和中断
- `SysTimer_*` 系列函数 (NMSIS Core API)

中断号固定为 **IRQ 7**（Machine Timer Interrupt）。

### BL616CL

BL616CL 变体使用 MCU_MISC 模块中的 E907 RTC 定时器：

| 寄存器偏移 | 用途 |
|-----------|------|
| `0x08` | RTC 加载值低 32 位 |
| `0x0C` | RTC 加载值高 32 位 |
| `0x14` | RTC 控制（bit 0: 使能, bit 1: 复位, bit 28: 加载脉冲） |

基地址: `BFLB_MISC_BASE = 0x20009000`

---

## 中断配置

MTIMER 使用 RISC-V 标准 Machine Timer 中断（IRQ 7）。

**中断注册示例：**

```c
void mtimer_handler(int irq, void *arg)
{
    // 定时器中断处理
    // 框架自动重置 mtimecmp 以维持周期性中断
}

// 配置定时器在 1000 ticks 后触发
bflb_mtimer_config(1000, mtimer_handler);
```

内部实现会自动：
1. 保存 ticks 值和回调函数
2. 禁用 IRQ 7
3. 设置 mtimecmp 比较值
4. 注册 ISR 回调（`bflb_irq_attach(7, systick_isr, NULL)`）
5. 使能 IRQ 7

在 ISR 中自动将比较值递增 `current_set_ticks`，实现周期性定时。

---

## 注意事项

1. **TCM 驻留:** `bflb_mtimer_get_time_us()`、`bflb_mtimer_delay_us()`、`bflb_mtimer_delay_ms()` 等函数标记为 `ATTR_TCM_SECTION`，驻留在 TCM（Tightly Coupled Memory）中以获得最低访问延迟。

2. **忙等待:** 延时函数采用忙等待（busy-wait），会持续占用 CPU。对于需要低功耗的场景，请考虑使用 RTC 定时器或硬件 Timer 外设。

3. **频率精度:** 默认 1 MHz 是近似值。若需精确时序，请测量实际频率并通过 `bflb_mtimer_get_freq()` 返回精确值，同时启用 `CONFIG_MTIMER_CUSTOM_FREQUENCE`。

4. **中断冲突:** 定时器中断使用 IRQ 7，请勿与其他外设共享此中断号。

5. **32 位安全:** 在 32 位 RISC-V 平台读取 64 位 `mtime` 寄存器时，库函数内部采用双读防翻转机制，确保读取一致性。
