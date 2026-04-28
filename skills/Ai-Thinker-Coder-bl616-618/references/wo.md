# WO Waveform Output API Reference (BL616/BL618)

## 概述

**WO (Waveform Output)** 是 BL616/BL618 芯片的波形输出外设，通过GPIO产生精确的双电平波形信号，广泛用于驱动 WS2812 等单线协议LED、UART bit-banging、信号调制等场景。

WO 实际上嵌入在 **GLB (Global Controller)** 外设内部，通过配置 GPIO 的特殊功能模式工作。核心原理：配置两个电平的时间比例（code0/code1），通过 FIFO 推送 16 位数据，每位决定输出 code0 还是 code1 波形。

**典型应用**：
- WS2812 / NeoPixel RGB LED 驱动
- 单线协议设备通信
- UART 透传（WO UART 模式）
- 任意双电平信号输出

**基地址**：WO 寄存器位于 GLB 地址空间，通过 `bflb_device_get_by_name("wo")` 获取设备句柄。

---

## 头文件

```c
#include "bflb_wo.h"           // LHAL API
#include "hardware/wo_reg.h"    // 寄存器定义
```

---

## 模式定义

### WO_MODE — 工作模式

```c
#define WO_MODE_WRITE   0  // WO direct write（直接写入）
#define WO_MODE_SET_CLR 1  // WO set/clr（设置/清除）
```

### WO_INT — 中断类型

```c
#define WO_INT_END  (1 << 0)  // WO 传输结束中断（TX FIFO 空）
#define WO_INT_FIFO (1 << 1)  // WO FIFO 就绪中断（可写入数据）
#define WO_INT_FER  (1 << 2)  // WO FIFO 错误中断（溢出/下溢）
```

---

## 配置结构体

### bflb_wo_cfg_s

```c
struct bflb_wo_cfg_s {
    uint16_t code_total_cnt;     // 一个周期的总计数，应小于 512
    uint8_t  code0_first_cnt;   // code0 高电平计数，应小于 256
    uint8_t  code1_first_cnt;   // code1 高电平计数，应小于 256
    uint8_t  code0_first_level; // code0 起始电平（0=低，1=高）
    uint8_t  code1_first_level; // code1 起始电平（0=低，1=高）
    uint8_t  idle_level;        // 空闲时 GPIO 电平（0=低，1=高）
    uint8_t  fifo_threshold;    // FIFO 阈值，应小于 128
    uint8_t  mode;              // 工作模式，使用 WO_MODE
};
```

---

## 寄存器映射（GLB GPIO 配置区）

WO 实际由 GLB 外设内的 GPIO 配置寄存器实现：

| 寄存器 | 地址偏移 | 说明 |
|--------|----------|------|
| `GLB_GPIO_CFG0 + pin×4` | `0x8C4 + pin×4` | 每个 GPIO 引脚的配置（含 WO 功能选择） |
| `GLB_GPIO_CFG142` | `0xAFC` | WO 核心控制：波形时序、FIFO、DMA |
| `GLB_GPIO_CFG143` | `0xB00` | WO DMA 使能、FIFO 状态、中断 |
| `GLB_GPIO_CFG144` | `0xB04` | WO 发送数据端口 |

### GLB_GPIO_CFG142 关键位字段

```c
// 0xAFC : GLB_GPIO_CFG142
#define GLB_CR_INVERT_CODE0_HIGH   (1 << 1)   // code0 电平反转
#define GLB_CR_INVERT_CODE1_HIGH   (1 << 2)   // code1 电平反转
#define GLB_CR_CODE_TOTAL_TIME_SHIFT (7)       // 周期总计数（9位）
#define GLB_CR_CODE_TOTAL_TIME_MASK  (0x1FF << 7)
#define GLB_CR_CODE0_HIGH_TIME_SHIFT (16)      // code0 高电平计数（8位）
#define GLB_CR_CODE0_HIGH_TIME_MASK (0xFF << 16)
#define GLB_CR_CODE1_HIGH_TIME_SHIFT (24)      // code1 高电平计数（8位）
#define GLB_CR_CODE1_HIGH_TIME_MASK (0xFF << 24)
#define GLB_CR_GPIO_TX_EN           (1 << 0)   // WO 输出使能
```

### GLB_GPIO_CFG143 关键位字段

```c
// 0xB00 : GLB_GPIO_CFG143
#define GLB_CR_GPIO_DMA_TX_EN       (1 << 0)   // WO DMA 发送使能
#define GLB_GPIO_TX_FIFO_CLR       (1 << 2)   // FIFO 清零
#define GLB_GPIO_TX_END_CLR        (1 << 3)   // 传输结束标志清零
#define GLB_GPIO_TX_FIFO_OVERFLOW   (1 << 4)   // FIFO 溢出标志（只读）
#define GLB_GPIO_TX_FIFO_UNDERFLOW  (1 << 5)   // FIFO 下溢标志（只读）
#define GLB_CR_GPIO_TX_FIFO_TH_SHIFT (16)      // FIFO 阈值（7位）
#define GLB_CR_GPIO_TX_END_MASK    (1 << 23)   // 结束中断屏蔽
#define GLB_CR_GPIO_TX_FIFO_MASK   (1 << 24)   // FIFO 中断屏蔽
#define GLB_CR_GPIO_TX_FER_MASK    (1 << 25)   // FIFO 错误中断屏蔽
#define GLB_R_GPIO_TX_END_INT      (1 << 26)   // 结束中断标志（只读）
#define GLB_R_GPIO_TX_FIFO_INT     (1 << 27)   // FIFO 中断标志（只读）
#define GLB_R_GPIO_TX_FER_INT      (1 << 28)   // FIFO 错误中断标志（只读）
#define GLB_CR_GPIO_TX_END_EN      (1 << 29)   // 结束中断使能
#define GLB_CR_GPIO_TX_FIFO_EN     (1 << 30)   // FIFO 中断使能
#define GLB_CR_GPIO_TX_FER_EN      (1 << 31)   // FIFO 错误中断使能
```

---

## API 函数

### bflb_wo_pin_init — 初始化 WO 引脚

```c
void bflb_wo_pin_init(struct bflb_device_s *dev, uint8_t pin, uint8_t mode);
```

将指定 GPIO 引脚配置为 WO 模式并分配给 WO 外设。

**参数**：
- `pin` — GPIO 引脚号（如 `10`）
- `mode` — `WO_MODE_WRITE` 或 `WO_MODE_SET_CLR`

**实现细节**：将 GPIO 的 `FUNC_SEL` 设置为 `0xB`，`MODE` 设置为 `2`（WRITE）或 `3`（SET_CLR）。

---

### bflb_wo_init — 初始化 WO 外设

```c
void bflb_wo_init(struct bflb_device_s *dev, struct bflb_wo_cfg_s *cfg);
```

初始化 WO 核心功能（波形时序、FIFO、极性）。

> ⚠️ 对于 **BL616CL**，此 API 会将时钟分频器重置为 1。如需自定义分频，调用 `bflb_wo_set_clk_div()`。

---

### bflb_wo_set_clk_div — 设置时钟分频（仅 BL616CL）

```c
void bflb_wo_set_clk_div(struct bflb_device_s *dev, uint16_t clk_div);
```

**注意**：
- 分频值默认为 1
- `bflb_wo_init()` 会将分频重置为 1，**需在 `bflb_wo_init()` 之后调用**
- 分频值非 1 时，WO 首次产生的波形占空比可能不准确

---

### bflb_wo_enable / bflb_wo_disable — 使能/关闭 WO

```c
void bflb_wo_enable(struct bflb_device_s *dev);
void bflb_wo_disable(struct bflb_device_s *dev);
```

使能后 GPIO 开始输出 WO 波形，关闭后 GPIO 恢复普通 GPIO 功能。

---

### bflb_wo_get_fifo_available_cnt — 查询 FIFO 可用空间

```c
uint32_t bflb_wo_get_fifo_available_cnt(struct bflb_device_s *dev);
```

**返回**：FIFO 当前可写入的 16 位数据个数。

---

### bflb_wo_push_fifo — 推送数据到 FIFO

```c
uint32_t bflb_wo_push_fifo(struct bflb_device_s *dev, uint16_t *data, uint32_t len);
```

**返回**：成功写入的数据个数（可能少于 `len`，取决于 FIFO 剩余空间）。

---

### bflb_wo_push_fifo_force — 强制推送数据

```c
void bflb_wo_push_fifo_force(struct bflb_device_s *dev, uint16_t *data, uint32_t len);
```

强制推送数据，**直到 FIFO 满才停止**（阻塞式）。

---

### bflb_wo_clear_fifo — 清空 FIFO

```c
void bflb_wo_clear_fifo(struct bflb_device_s *dev);
```

---

### bflb_wo_enable_dma / bflb_wo_disable_dma — DMA 使能

```c
void bflb_wo_enable_dma(struct bflb_device_s *dev);
void bflb_wo_disable_dma(struct bflb_device_s *dev);
```

WO 支持 DMA 传输，DMA 源为 `DMA_REQUEST_WO`。

---

### 中断相关

```c
uint32_t bflb_wo_get_int_status(struct bflb_device_s *dev);  // 获取中断状态
void bflb_wo_int_mask(struct bflb_device_s *dev, uint32_t int_type);      // 屏蔽中断
void bflb_wo_int_unmask(struct bflb_device_s *dev, uint32_t int_type);   // 解除屏蔽
void bflb_wo_int_clear(struct bflb_device_s *dev, uint32_t int_type);    // 清除中断标志
```

---

### WO UART 模式

WO 内置 UART bit-banging 模式，可通过任意 GPIO 模拟 UART 发送：

```c
// 初始化 WO UART
void bflb_wo_uart_init(struct bflb_device_s *dev, uint32_t baudrate, uint8_t pin);

// 发送单个字符
void bflb_wo_uart_putchar(struct bflb_device_s *dev, uint8_t ch);

// 发送数据块（轮询）
void bflb_wo_uart_put(struct bflb_device_s *dev, uint8_t *data, uint32_t len);
```

---

## WS2812 驱动完整示例

WS2812 是单线RGB LED，每个 bit 由一个周期表示：
- `0` = 高电平 0.4μs + 低电平 0.85μs
- `1` = 高电平 0.85μs + 低电平 0.4μs
- 每个 RGB 像素需要 24 个 bit（GRB 顺序）

```c
#include "bflb_wo.h"
#include "bflb_dma.h"
#include "bflb_mtimer.h"

#define WS2812_PIN    10
#define WS2812_NUM    60
#define WS2812_BUFFER (WS2812_NUM * 24)

struct bflb_device_s *wo;
struct bflb_device_s *dma0_ch0;
static ATTR_NOCACHE_RAM_SECTION struct bflb_dma_channel_lli_pool_s llipool[1];
static ATTR_NOCACHE_RAM_SECTION struct bflb_dma_channel_lli_transfer_s transfers[1];
uint16_t buffer_data[WS2812_BUFFER] __attribute__((aligned(32)));

/* WS2812 时序配置（XCLK=40MHz，周期=1.25μs） */
struct bflb_wo_cfg_s wo_cfg = {
    .code_total_cnt = 50,   /* 40MHz / 50 = 800kHz */
    .code0_first_cnt = 16,  /* 高电平 0.4μs = 1.25μs * 16/50 */
    .code1_first_cnt = 34,  /* 高电平 0.85μs = 1.25μs * 34/50 */
    .code0_first_level = 1,
    .code1_first_level = 1,
    .idle_level = 0,
    .fifo_threshold = 64,
    .mode = WO_MODE_WRITE,
};

struct bflb_dma_channel_config_s dma_cfg = {
    .direction = DMA_MEMORY_TO_PERIPH,
    .src_req = DMA_REQUEST_NONE,
    .dst_req = DMA_REQUEST_WO,
    .src_addr_inc = DMA_ADDR_INCREMENT_ENABLE,
    .dst_addr_inc = DMA_ADDR_INCREMENT_DISABLE,
    .src_burst_count = DMA_BURST_INCR8,
    .dst_burst_count = DMA_BURST_INCR8,
    .src_width = DMA_DATA_WIDTH_16BIT,
    .dst_width = DMA_DATA_WIDTH_16BIT,
};

/* 将 RGB 颜色转换为 WS2812 数据（GRB 顺序） */
static void set_rgb_color(uint16_t index, uint8_t r, uint8_t g, uint8_t b)
{
    for (int i = 0; i < 8; i++) {
        buffer_data[index * 24 + i]     = (g & (0x80 >> i)) ? 34 : 16;  /* G */
        buffer_data[index * 24 + 8 + i] = (r & (0x80 >> i)) ? 34 : 16;  /* R */
        buffer_data[index * 24 + 16 + i]= (b & (0x80 >> i)) ? 34 : 16;  /* B */
    }
}

void app_main(void)
{
    /* 获取设备句柄 */
    wo = bflb_device_get_by_name("wo");
    dma0_ch0 = bflb_device_get_by_name("dma0_ch0");

    /* 初始化 WO 引脚 */
    bflb_wo_pin_init(wo, WS2812_PIN, WO_MODE_WRITE);

    /* 初始化 WO */
    bflb_wo_init(wo, &wo_cfg);

    /* 配置 DMA */
    bflb_dma_channel_init(dma0_ch0, DMA_CH0, &dma_cfg);
    bflb_dma_channel_lli_reinit(dma0_ch0, DMA_CH0, llipool, 1);

    /* 填充颜色数据 */
    for (int i = 0; i < WS2812_NUM; i++) {
        set_rgb_color(i, 255, 0, 0);  /* 全红 */
    }

    /* 准备 DMA 传输 */
    transfers[0].src_addr = (uint32_t)buffer_data;
    transfers[0].dst_addr = (uint32_t)(dev->reg_base + 0xB04);  /* GLB_GPIO_CFG144 */
    transfers[0].next = (uint32_t)0;
    transfers[0].nbytes = sizeof(buffer_data);
    bflb_dma_channel_lli_add_node(dma0_ch0, DMA_CH0, transfers);

    /* 启动 DMA + WO */
    bflb_wo_enable_dma(wo);
    bflb_wo_enable(wo);
    bflb_dma_channel_start(dma0_ch0, DMA_CH0);

    /* 等待传输完成 */
    bflb_mtimer_delay_ms(2);  /* WS2812 需要 >50μs 低电平复位 */
}
```

---

## WO UART 模式示例（GPIO 模拟 UART）

```c
#include "bflb_wo.h"

struct bflb_device_s *wo;

void app_main(void)
{
    wo = bflb_device_get_by_name("wo");

    /* 将 GPIO10 配置为 WO UART，波特率 115200 */
    bflb_wo_uart_init(wo, 115200, 10);

    /* 发送字符串 */
    bflb_wo_uart_put(wo, (uint8_t *)"Hello WO UART\r\n", 15);
}
```

---

## 寄存器级编程

直接操作 GLB 寄存器配置 WO：

```c
#include "hardware/wo_reg.h"

#define GLB_BASE  0x20000000
#define GLB_GPIO_CFG142_OFFSET  0xAFC

/* 假设已通过 bflb_gpio_init 将引脚配置为 WO 功能 */
uint32_t reg_base = GLB_BASE;

/* 配置波形时序 */
uint32_t regval = getreg32(reg_base + GLB_GPIO_CFG142_OFFSET);
regval &= ~GLB_CR_CODE_TOTAL_TIME_MASK;
regval &= ~GLB_CR_CODE0_HIGH_TIME_MASK;
regval &= ~GLB_CR_CODE1_HIGH_TIME_MASK;
regval |= (50 << GLB_CR_CODE_TOTAL_TIME_SHIFT);
regval |= (16 << GLB_CR_CODE0_HIGH_TIME_SHIFT);   /* code0 = 16/50 高电平 */
regval |= (34 << GLB_CR_CODE1_HIGH_TIME_SHIFT);   /* code1 = 34/50 高电平 */
putreg32(regval, reg_base + GLB_GPIO_CFG142_OFFSET);

/* 使能 WO */
regval = getreg32(reg_base + GLB_GPIO_CFG142_OFFSET);
regval |= GLB_CR_GPIO_TX_EN;
putreg32(regval, reg_base + GLB_GPIO_CFG142_OFFSET);

/* 写数据到 FIFO（通过 GLB_GPIO_CFG144 = 0xB04）*/
for (int i = 0; i < len; i++) {
    putreg32(data[i] & 0xFFFF, reg_base + 0xB04);
}
```

---

## SDK 示例路径

| 示例 | 说明 |
|------|------|
| `examples/peripherals/wo/wo_ws2812/` | WS2812 RGB LED 驱动 + DMA |
| `examples/peripherals/wo/wo_console/` | WO UART Console |
| `examples/peripherals/wo/wo_int/` | WO 中断示例 |
| `examples/peripherals/wo/wo_uart/` | WO UART 模式 |
| `examples/peripherals/wo/wo_dma/` | WO DMA 传输 |

---

## 注意事项

1. **XCLK 时钟**：WO 依赖 `XCLK` 时钟源，初始化前需确保时钟已使能
2. **WS2812 时序**：必须严格遵守 WS2812 规格（800kHz ±15%），XCLK=40MHz 时 `code_total_cnt=50` 正好是 800kHz
3. **DMA 传输**：DMA 目标地址固定为 `GLB_BASE + 0xB04`（GLB_GPIO_CFG144）
4. **FIFO 阈值**：`fifo_threshold` 应设为 64 以下，否则可能触发 FIFO 错误中断
5. **多字节颜色**：WS2812 使用 **GRB** 顺序，不是常见的 RGB
6. **复位时间**：发送完数据后需要至少 **50μs 低电平** 复位才能发送到下一个 LED
