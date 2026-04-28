# DPI API Reference (BL616/BL618)

> **Source:** `bouffalo_sdk/drivers/lhal/include/bflb_dpi.h`  
> **Implementation:** `bouffalo_sdk/drivers/lhal/src/bflb_dpi.c`  
> **Register Headers:** `hardware/mm_misc_reg.h`, `hardware/dvp_tsrc_reg.h`  

## Overview

DPI（Display Parallel Interface，显示并行接口）模块用于驱动 RGB/MIPI-DPI 接口的 LCD 显示屏。它将帧缓冲区（framebuffer）中的像素数据通过并行数据线输出到显示面板，支持多种色彩格式和时序配置。

该模块内部包含 DTSRC（Data Transport Source）子系统，通过 AXI 总线读取帧缓冲区数据，并按照 DPI 时序输出。支持帧缓冲切换（双缓冲）、测试图案输出以及 OSD（On-Screen Display）叠加。

> **注意：** DPI 模块仅支持 BL618DG 芯片。

## Base Address

| 子系统 | Base Address | 说明 |
|--------|-------------|------|
| MM_MISC (显示配置) | `dev->reg_base` | DPI 时序、接口配置、Y2R/R2Y 色彩转换 |
| DTSRC (数据传输) | `0x20045000` | 帧缓冲读取、像素输出、测试图案 |

---

## 宏定义

### 输入选择 (DPI_INPUT_SEL)

| 宏 | 值 | 说明 |
|----|---|------|
| `DPI_INPUT_SEL_TEST_PATTERN_WITHOUT_OSD` | 0 | 测试图案输入（无 OSD） |
| `DPI_INPUT_SEL_TEST_PATTERN_WITH_OSD` | 1 | 测试图案输入（带 OSD） |
| `DPI_INPUT_SEL_FRAMEBUFFER_WITHOUT_OSD` | 2 | 帧缓冲输入（无 OSD），仅支持 RGB 格式 |
| `DPI_INPUT_SEL_FRAMEBUFFER_WITH_OSD` | 3 | 帧缓冲输入（带 OSD），支持 RGB 和 YUV 格式 |

### 接口色彩编码 (DPI_INTERFACE)

| 宏 | 值 | 说明 |
|----|---|------|
| `DPI_INTERFACE_24_PIN` | 0 | 24 引脚接口 (D0–D23) |
| `DPI_INTERFACE_18_PIN_MODE_1` | 1 | 18 引脚接口 Mode 1 (D0–D17) |
| `DPI_INTERFACE_18_PIN_MODE_2` | 2 | 18 引脚接口 Mode 2 (D0–D5, D8–D13, D16–D21) |
| `DPI_INTERFACE_16_PIN_MODE_1` | 3 | 16 引脚接口 Mode 1 (D0–D15) |
| `DPI_INTERFACE_16_PIN_MODE_2` | 4 | 16 引脚接口 Mode 2 (D0–D4, D8–D13, D16–D20) |
| `DPI_INTERFACE_16_PIN_MODE_3` | 5 | 16 引脚接口 Mode 3 (D1–D5, D8–D13, D17–D21) |

### 测试图案颜色 (DPI_TEST)

| 宏 | 值 | 说明 |
|----|---|------|
| `DPI_TEST_PATTERN_NULL` | 0 | 无测试图案 |
| `DPI_TEST_PATTERN_BLACK` | 1 | 黑色 |
| `DPI_TEST_PATTERN_RED` | 2 | 红色 |
| `DPI_TEST_PATTERN_GREE` | 3 | 绿色 |
| `DPI_TEST_PATTERN_YELLOW` | 4 | 黄色 |

### 数据格式 (DPI_DATA)

| 宏 | 值 | 说明 |
|----|---|------|
| `DPI_DATA_FORMAT_YUYV` | 0 | YUYV 交错格式 [31:24]=V, [23:16]=Y, [15:8]=U, [7:0]=Y |
| `DPI_DATA_FORMAT_RGB888` | 1 | 24-bit RGB [31:24]=B, [23:16]=R, [15:8]=G, [7:0]=B |
| `DPI_DATA_FORMAT_RGB565` | 2 | 16-bit RGB565 |
| `DPI_DATA_FORMAT_NRGB8888` | 3 | 32-bit NRGB8888 [31:24]=N, [23:16]=R, [15:8]=G, [7:0]=B |
| `DPI_DATA_FORMAT_Y_UV_PLANAR` | 4 | Y/UV 平面分离格式（Y 和 UV 分量独立存储） |
| `DPI_DATA_FORMAT_UYVY` | 5 | UYVY 交错格式 [31:24]=Y, [23:16]=V, [15:8]=Y, [7:0]=U |
| `DPI_DATA_FORMAT_BGR888` | 6 | 24-bit BGR [31:24]=R, [23:16]=B, [15:8]=G, [7:0]=R |
| `DPI_DATA_FORMAT_BGR565` | 7 | 16-bit BGR565 |
| `DPI_DATA_FORMAT_NBGR8888` | 8 | 32-bit NBGR8888 |
| `DPI_DATA_FORMAT_TEST_PATTERN` | 10 | 测试图案格式 |

### YUV420 UV 有效行 (DPI_YUV420)

| 宏 | 值 | 说明 |
|----|---|------|
| `DPI_YUV420_UV_VALID_LINE_EVEN` | 0 | 偶数行 UV 有效 (0/2/4/6/8...) |
| `DPI_YUV420_UV_VALID_LINE_ODD` | 1 | 奇数行 UV 有效 (1/3/5/7/9...) |

### 突发传输长度 (DPI_BURST)

| 宏 | 值 | 说明 |
|----|---|------|
| `DPI_BURST_INCR1` | 0 | 突发长度 1 |
| `DPI_BURST_INCR4` | 1 | 突发长度 4 |
| `DPI_BURST_INCR8` | 2 | 突发长度 8 |
| `DPI_BURST_INCR16` | 3 | 突发长度 16 |
| `DPI_BURST_INCR32` | 5 | 突发长度 32 |
| `DPI_BURST_INCR64` | 6 | 突发长度 64 |

### Feature Control 命令 (DPI_CMD)

| 宏 | 值 | 说明 |
|----|---|------|
| `DPI_CMD_SET_YUV420_UV_VALID_LINE` | 0x01 | 设置 YUV420 UV 有效行 |
| `DPI_CMD_SET_BURST` | 0x02 | 设置 AXI 突发传输长度 |

---

## 数据结构

### bflb_dpi_config_s

DPI 配置结构体。

```c
struct bflb_dpi_config_s {
    uint16_t width;               // 有效像素宽度
    uint16_t height;              // 有效像素高度
    uint8_t hsw;                  // 水平同步宽度
    uint16_t hbp;                 // 水平后廊
    uint8_t hfp;                  // 水平前廊
    uint8_t vsw;                  // 垂直同步宽度
    uint16_t vbp;                 // 垂直后廊
    uint8_t vfp;                  // 垂直前廊
    uint8_t interface;            // 接口色彩编码, 使用 DPI_INTERFACE 宏
    uint8_t input_sel;            // 输入选择, 使用 DPI_INPUT_SEL 宏
    uint8_t test_pattern;         // 测试图案颜色, 使用 DPI_TEST 宏
    uint8_t data_format;          // 数据格式, 使用 DPI_DATA 宏
    uint32_t framebuffer_addr;    // 帧缓冲区起始地址 (Y_UV_PLANAR 模式下为 Y 帧缓冲地址)
    uint32_t uv_framebuffer_addr; // UV 帧缓冲区起始地址 (仅 Y_UV_PLANAR 模式)
};
```

---

## LHAL API 函数

### bflb_dpi_init

初始化 DPI 接口，配置时序、数据格式和输入源。

```c
void bflb_dpi_init(struct bflb_device_s *dev, const struct bflb_dpi_config_s *config);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `config` | `const struct bflb_dpi_config_s *` | DPI 配置结构体指针 |

---

### bflb_dpi_enable

使能 DPI 输出。

```c
void bflb_dpi_enable(struct bflb_device_s *dev);
```

---

### bflb_dpi_disable

禁用 DPI 输出。

```c
void bflb_dpi_disable(struct bflb_device_s *dev);
```

---

### bflb_dpi_framebuffer_switch

切换帧缓冲区地址（双缓冲切换）。

```c
void bflb_dpi_framebuffer_switch(struct bflb_device_s *dev, uint32_t addr);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `addr` | `uint32_t` | 新的帧缓冲区起始地址 |

---

### bflb_dpi_framebuffer_planar_switch

切换 Y 和 UV 帧缓冲区地址（仅 `DPI_DATA_FORMAT_Y_UV_PLANAR` 模式）。

```c
void bflb_dpi_framebuffer_planar_switch(struct bflb_device_s *dev, uint32_t y_addr, uint32_t uv_addr);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `y_addr` | `uint32_t` | 新的 Y 帧缓冲区起始地址 |
| `uv_addr` | `uint32_t` | 新的 UV 帧缓冲区起始地址 |

---

### bflb_dpi_get_framebuffer_using

获取当前正在使用的帧缓冲区地址。

```c
uint32_t bflb_dpi_get_framebuffer_using(struct bflb_device_s *dev);
```

**返回值:** 当前使用的帧缓冲区起始地址。

---

### bflb_dpi_set_test_pattern_custom

设置 DPI 自定义测试图案颜色参数。

```c
void bflb_dpi_set_test_pattern_custom(struct bflb_device_s *dev, uint16_t max, uint16_t value, uint8_t step);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `max` | `uint16_t` | 颜色最大值 |
| `value` | `uint16_t` | 颜色起始值 |
| `step` | `uint8_t` | 颜色步进值 |

---

### bflb_dpi_feature_control

DPI 特性控制接口。

```c
int bflb_dpi_feature_control(struct bflb_device_s *dev, int cmd, size_t arg);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `cmd` | `int` | 特性命令, 使用 DPI_CMD 宏 |
| `arg` | `size_t` | 命令参数 |

**返回值:** 成功返回 0，失败返回 `-EPERM`。

---

## 使用示例

### 示例 1: 基本 RGB888 DPI 显示

```c
#include "bflb_dpi.h"

#define LCD_WIDTH  480
#define LCD_HEIGHT 272

static uint32_t framebuffer[LCD_WIDTH * LCD_HEIGHT] __attribute__((aligned(16)));
static uint32_t framebuffer_back[LCD_WIDTH * LCD_HEIGHT] __attribute__((aligned(16)));

void dpi_lcd_example(void)
{
    struct bflb_device_s *dpi;

    dpi = bflb_device_get_by_name("dpi");

    struct bflb_dpi_config_s config = {
        .width = LCD_WIDTH,
        .height = LCD_HEIGHT,
        .hsw = 1,          // 水平同步宽度
        .hbp = 40,         // 水平后廊
        .hfp = 5,          // 水平前廊
        .vsw = 1,          // 垂直同步宽度
        .vbp = 8,          // 垂直后廊
        .vfp = 8,          // 垂直前廊
        .interface = DPI_INTERFACE_24_PIN,
        .input_sel = DPI_INPUT_SEL_FRAMEBUFFER_WITHOUT_OSD,
        .test_pattern = DPI_TEST_PATTERN_NULL,
        .data_format = DPI_DATA_FORMAT_NRGB8888,
        .framebuffer_addr = (uint32_t)framebuffer,
        .uv_framebuffer_addr = 0,
    };

    bflb_dpi_init(dpi, &config);
    bflb_dpi_enable(dpi);

    // 填充帧缓冲为红色
    for (int i = 0; i < LCD_WIDTH * LCD_HEIGHT; i++) {
        framebuffer[i] = 0x00FF0000;  // NRGB8888: N=0, R=0xFF, G=0, B=0
    }
}
```

### 示例 2: 双缓冲切换显示

```c
void dpi_double_buffer_example(void)
{
    struct bflb_device_s *dpi = bflb_device_get_by_name("dpi");

    // ... 初始化 DPI 配置 ...

    // 先填充后缓冲
    for (int i = 0; i < LCD_WIDTH * LCD_HEIGHT; i++) {
        framebuffer_back[i] = 0x0000FF00;  // 绿色
    }

    // 切换到后缓冲 (无缝切换)
    bflb_dpi_framebuffer_switch(dpi, (uint32_t)framebuffer_back);
}
```

### 示例 3: 测试图案输出

```c
void dpi_test_pattern_example(void)
{
    struct bflb_device_s *dpi = bflb_device_get_by_name("dpi");

    struct bflb_dpi_config_s config = {
        .width = 480,
        .height = 272,
        .hsw = 1,
        .hbp = 40,
        .hfp = 5,
        .vsw = 1,
        .vbp = 8,
        .vfp = 8,
        .interface = DPI_INTERFACE_24_PIN,
        .input_sel = DPI_INPUT_SEL_TEST_PATTERN_WITHOUT_OSD,
        .test_pattern = DPI_TEST_PATTERN_RED,
        .data_format = DPI_DATA_FORMAT_TEST_PATTERN,
        .framebuffer_addr = 0,
        .uv_framebuffer_addr = 0,
    };

    bflb_dpi_init(dpi, &config);
    bflb_dpi_enable(dpi);
}
```

### 示例 4: 自定义测试图案

```c
void dpi_custom_test_pattern_example(void)
{
    struct bflb_device_s *dpi = bflb_device_get_by_name("dpi");

    struct bflb_dpi_config_s config = {
        .width = 480,
        .height = 272,
        .hsw = 1,
        .hbp = 40,
        .hfp = 5,
        .vsw = 1,
        .vbp = 8,
        .vfp = 8,
        .interface = DPI_INTERFACE_24_PIN,
        .input_sel = DPI_INPUT_SEL_TEST_PATTERN_WITHOUT_OSD,
        .test_pattern = DPI_TEST_PATTERN_NULL,
        .data_format = DPI_DATA_FORMAT_TEST_PATTERN,
        .framebuffer_addr = 0,
        .uv_framebuffer_addr = 0,
    };

    bflb_dpi_init(dpi, &config);

    // 设置自定义渐变色: 最大值 0xFFFF, 起始值 0, 步进 4
    bflb_dpi_set_test_pattern_custom(dpi, 0xFFFF, 0, 4);

    bflb_dpi_enable(dpi);
}
```

### 示例 5: Y_UV_PLANAR 格式 + AXI 突发配置

```c
void dpi_yuv_planar_example(void)
{
    struct bflb_device_s *dpi = bflb_device_get_by_name("dpi");

    struct bflb_dpi_config_s config = {
        .width = 480,
        .height = 272,
        .hsw = 1,
        .hbp = 40,
        .hfp = 5,
        .vsw = 1,
        .vbp = 8,
        .vfp = 8,
        .interface = DPI_INTERFACE_24_PIN,
        .input_sel = DPI_INPUT_SEL_FRAMEBUFFER_WITHOUT_OSD,
        .data_format = DPI_DATA_FORMAT_Y_UV_PLANAR,
        .framebuffer_addr = (uint32_t)y_buffer,
        .uv_framebuffer_addr = (uint32_t)uv_buffer,
    };

    bflb_dpi_init(dpi, &config);

    // 设置 AXI 突发长度为 16
    bflb_dpi_feature_control(dpi, DPI_CMD_SET_BURST, DPI_BURST_INCR16);

    // 设置 YUV420 UV 有效行为奇数行
    bflb_dpi_feature_control(dpi, DPI_CMD_SET_YUV420_UV_VALID_LINE,
                             DPI_YUV420_UV_VALID_LINE_ODD);

    bflb_dpi_enable(dpi);

    // 切换 Y/UV 缓冲区
    bflb_dpi_framebuffer_planar_switch(dpi,
                                       (uint32_t)y_buffer_back,
                                       (uint32_t)uv_buffer_back);
}
```

---

## 寄存器信息

DPI 模块涉及两个寄存器组。

### MM_MISC 寄存器 (显示配置)

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `MM_MISC_DISP_CONFIG` | `0x300` | 显示配置（含 DPI 使能和接口选择） |
| `MM_MISC_DISP_DPI_CONFIG` | `0x304` | DPI 时序配置 |
| `MM_MISC_DVP_MUX_SEL_REG2` | `0x14` | DVP 复用选择（含 MUXO 和 OSD 选择） |
| `MM_MISC_DISP_Y2R_CONFIG_0–7` | `0x200–0x21C` | YUV→RGB 色彩空间转换矩阵 |
| `MM_MISC_DISP_R2Y_CONFIG_0–7` | `0x240–0x25C` | RGB→YUV 色彩空间转换矩阵 |

#### DISP_CONFIG 寄存器 (偏移 0x300)

| 位 | 字段 | 说明 |
|----|------|------|
| 1 | `RG_DISP_DPI_EN` | DPI 使能 (1=使能) |
| 4–6 | `RG_DISP_DPI_ICC` | DPI 接口色彩编码选择 |

#### DISP_DPI_CONFIG 寄存器 (偏移 0x304)

| 位 | 字段 | 说明 |
|----|------|------|
| 0–7 | `RG_DISP_DPI_HS_W` | 水平同步宽度 (pixel clock) |
| 8–15 | `RG_DISP_DPI_HFP_W` | 水平前廊宽度 (pixel clock) |
| 16–23 | `RG_DISP_DPI_VS_W` | 垂直同步宽度 (line count) |
| 24–31 | `RG_DISP_DPI_VFP_W` | 垂直前廊宽度 (line count) |

> **注意：** 水平后廊 HBP 和垂直后廊 VBP 通过 DTSRC 的 `FRAME_SIZE_H`/`FRAME_SIZE_V` 隐式计算：  
> `BLANK_H = HSW + HBP + HFP`  
> `BLANK_V = VSW + VBP + VFP`

#### DVP_MUX_SEL_REG2 寄存器 (偏移 0x14)

| 位 | 字段 | 说明 |
|----|------|------|
| 0–1 | `RG_DISP_OSD_SEL` | OSD 源选择 |
| 4–5 | `RG_DISP_MUXO_SEL` | 显示 MUXO 源选择 (2=test pattern/dtsrc, 1=OSD) |

### DTSRC 寄存器 (Base: 0x20045000)

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `DVP_TSRC_CONFIG` | `0x00` | DTSRC 主配置寄存器 |
| `DVP_TSRC_FRAME_SIZE_H` | `0x04` | 水平帧尺寸（总宽度 + 空白宽度） |
| `DVP_TSRC_FRAME_SIZE_V` | `0x08` | 垂直帧尺寸（总高度 + 空白高度） |
| `DVP_TSRC_AXI2DVP_SETTING` | `0x0C` | AXI→DVP 转换设置 |
| `DVP_TSRC_PIX_DATA_RANGE` | `0x10` | 测试图案像素数据范围 |
| `DVP_TSRC_PIX_DATA_STEP` | `0x14` | 测试图案像素步进 |
| `DVP_TSRC_AXI2DVP_START_ADDR_BY` | `0x2C` | AXI 起始地址 BY |
| `DVP_TSRC_AXI2DVP_SWAP_ADDR_BY` | `0x30` | AXI 交换地址 BY |
| `DVP_TSRC_AXI2DVP_PREFETCH` | `0x34` | 预取行数设置 |
| `DVP_TSRC_AXI2DVP_START_ADDR_UV` | `0x40` | AXI 起始地址 UV |
| `DVP_TSRC_AXI2DVP_SWAP_ADDR_UV` | `0x44` | AXI 交换地址 UV |
| `DVP_TSRC_AXI_PUSH_MODE` | `0x60` | Push 模式控制 |

#### DVP_TSRC_CONFIG 寄存器 (偏移 0x00)

| 位 | 字段 | 说明 |
|----|------|------|
| 0 | `CR_ENABLE` | DTSRC 使能 |
| 1 | `CR_AXI_EN` | AXI 读取使能 |
| 3 | `CR_AXI_PUSH_MODE` | Push 模式 |
| 7 | `CR_AXI_SWAP_MODE` | 交换模式使能 |
| 8–11 | `CR_AXI_SWAP_IDX_SEL` | 交换索引选择 |
| 12 | `CR_AXI_SWAP_IDX_SWM` | 交换索引写模式 |
| 13 | `CR_AXI_SWAP_IDX_SWV` | 交换索引读/写状态 |
| 16–18 | `CR_AXI_DVP_DATA_MODE` | AXI→DVP 数据模式 |
| 20–21 | `CR_AXI_B0_SEL` | 字节 0 (B0) 选择 |
| 22–23 | `CR_AXI_B1_SEL` | 字节 1 (B1) 选择 |
| 24–25 | `CR_AXI_B2_SEL` | 字节 2 (B2) 选择 |

#### DVP_TSRC_FRAME_SIZE_H 寄存器 (偏移 0x04)

| 位 | 字段 | 说明 |
|----|------|------|
| 0–15 | 总宽度 | `HSW + HBP + HFP + Width`（低 16 位） |
| 16–31 | `CR_BLANK_H` | 水平空白总宽度 `HSW + HBP + HFP` |

#### DVP_TSRC_FRAME_SIZE_V 寄存器 (偏移 0x08)

| 位 | 字段 | 说明 |
|----|------|------|
| 0–15 | 总高度 | `VSW + VBP + VFP + Height`（低 16 位） |
| 16–31 | `CR_BLANK_V` | 垂直空白总宽度 `VSW + VBP + VFP` |

#### DVP_TSRC_AXI2DVP_SETTING 寄存器 (偏移 0x0C)

| 位 | 字段 | 说明 |
|----|------|------|
| 0–2 | `CR_AXI_XLEN` | AXI 突发长度 (0=1, 1=4, 2=8, 3=16, 5=32, 6=64) |
| 7 | `CR_AXI_422_SP_MODE` | YUV422 semi-planar 模式 |
| 8 | `CR_AXI_420_SP_MODE` | YUV420 semi-planar 模式 |
| 9 | `CR_AXI_420_UD_SEL` | YUV420 UV 行选择 (0=偶数行, 1=奇数行) |
