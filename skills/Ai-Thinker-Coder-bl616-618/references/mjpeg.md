# MJPEG API Reference (BL616/BL618)

> **Source:** `bouffalo_sdk/drivers/lhal/include/bflb_mjpeg.h`  
> **Implementation:** `bouffalo_sdk/drivers/lhal/src/bflb_mjpeg.c`  
> **Register Header:** `hardware/mjpeg_reg.h`  

## Overview

MJPEG（Motion JPEG）硬件编解码器模块提供 YUV→JPEG 压缩功能。该模块支持多种 YUV 输入格式（交错/平面），可通过硬件自动完成 JPEG 编码，输出标准的 JPEG 码流。适用于摄像头视频流编码、图像压缩等场景。

模块支持两种工作模式：
- **摄像头模式** (`bflb_mjpeg_start`/`bflb_mjpeg_stop`): 与 DVP 摄像头接口联动，实时压缩摄像头帧
- **软件模式** (`bflb_mjpeg_sw_run`): 从指定内存缓冲区读取 YUV 数据并编码
  - **标准模式**: 压缩指定帧数
  - **Kick 模式**: 手动控制逐块压缩，适合实时流控

每个模块最多缓存 **4 帧** 压缩后的 JPEG 数据 (`MJPEG_MAX_FRAME_COUNT`)。

---

## 宏定义

### 像素格式 (MJPEG_FORMAT)

| 宏 | 值 | 说明 |
|----|---|------|
| `MJPEG_FORMAT_YUV422_YUYV` | 0 | YUYV 交错格式 |
| `MJPEG_FORMAT_YUV422_YVYU` | 1 | YVYU 交错格式 |
| `MJPEG_FORMAT_YUV422_UYVY` | 2 | UYVY 交错格式 |
| `MJPEG_FORMAT_YUV422_VYUY` | 3 | VYUY 交错格式 |
| `MJPEG_FORMAT_YUV422SP_NV16` | 4 | YUV422 半平面 NV16 (U 在前) |
| `MJPEG_FORMAT_YUV422SP_NV61` | 5 | YUV422 半平面 NV61 (V 在前) |
| `MJPEG_FORMAT_YUV420SP_NV12` | 6 | YUV420 半平面 NV12 (U 在前) |
| `MJPEG_FORMAT_YUV420SP_NV21` | 7 | YUV420 半平面 NV21 (V 在前) |
| `MJPEG_FORMAT_GRAY` | 8 | 灰度（仅 Y 分量） |

### 中断状态 (MJPEG_INTSTS)

| 宏 | 值 | 说明 |
|----|---|------|
| `MJPEG_INTSTS_ONE_FRAME` | `(1 << 4)` | 一帧编码完成中断 |
| `MJPEG_INTSTS_KICK_DONE` | `(1 << 23)` | Kick 编码完成中断 (仅 BL616CL) |
| `MJPEG_INTSTS_SWAP` | `(1 << 30)` | 交换缓冲区中断 |

### 中断清除 (MJPEG_INTCLR)

| 宏 | 值 | 说明 |
|----|---|------|
| `MJPEG_INTCLR_ONE_FRAME` | `(1 << 8)` | 清除一帧完成中断 |
| `MJPEG_INTCLR_KICK_DONE` | `(1 << 5)` | 清除 Kick 完成中断 (仅 BL616CL) |
| `MJPEG_INTCLR_SWAP` | `(1 << 13)` | 清除交换中断 |

### Feature Control 命令 (MJPEG_CMD)

| 宏 | 值 | 说明 |
|----|---|------|
| `MJPEG_CMD_SET_INPUTADDR0` | 0x00 | 设置输入缓冲区地址 0（YY 帧地址） |
| `MJPEG_CMD_SET_INPUTADDR1` | 0x01 | 设置输入缓冲区地址 1（UV 帧地址） |
| `MJPEG_CMD_SET_KICK_DONE_DELAY` | 0x02 | 设置 Kick 完成延迟 (仅 BL616CL) |
| `MJPEG_CMD_UPDATE_KICK_ADDR` | 0x03 | 更新 Kick 地址 (仅 BL616CL) |
| `MJPEG_CMD_READ_HW_VERSION` | 0x04 | 读取硬件版本 (仅 BL616CL) |
| `MJPEG_CMD_READ_SW_USAGE` | 0x05 | 读取软件使用标志 (仅 BL616CL) |
| `MJPEG_CMD_WRITE_SW_USAGE` | 0x06 | 写入软件使用标志 (仅 BL616CL) |
| `MJPEG_CMD_SWAP_ENABLE` | 0x07 | 使能/禁用 Swap 交换模式 |

### 参数校验宏

| 宏 | 说明 |
|----|------|
| `IS_MJPEG_FORMAT(type)` | 校验格式是否合法 (`<= MJPEG_FORMAT_GRAY`) |
| `IS_MJPEG_RESOLUTION(type)` | 校验分辨率是否为 8 的倍数 |
| `IS_MJPEG_QUALITY(type)` | 校验质量是否 ≤ 100 |
| `IS_MJPEG_ADDR(type)` | 校验地址是否 16 字节对齐 |

### 帧缓存常量

| 宏 | 值 | 说明 |
|----|---|------|
| `MJPEG_MAX_FRAME_COUNT` | 4 | 最大缓存帧数 |

---

## 数据结构

### bflb_mjpeg_config_s

MJPEG 配置结构体。

```c
struct bflb_mjpeg_config_s {
    uint8_t   format;           // 像素格式, 使用 MJPEG_FORMAT 宏
    uint8_t   quality;          // JPEG 压缩质量 (0–100)
    uint16_t  rows;             // 输入数据总行数 (用于计算 mem_hblk 行块数)
    uint16_t  resolution_x;     // 图像宽度 (必须是 8 或 16 的倍数)
    uint16_t  resolution_y;     // 图像高度 (必须是 8 或 16 的倍数)
    uint32_t  input_bufaddr0;   // 输入缓冲区 0 地址 (YY), 必须 16 字节对齐
    uint32_t  input_bufaddr1;   // 输入缓冲区 1 地址 (UV), 必须 16 字节对齐
    uint32_t  output_bufaddr;   // 输出 JPEG 缓冲区地址, 必须 16 字节对齐
    uint32_t  output_bufsize;   // 输出缓冲区大小 (应 > resolution_x * resolution_y * 2 * MJPEG_MAX_FRAME_COUNT)
    uint16_t *input_yy_table;   // 自定义 Y 量化表指针 (NULL 则使用默认)
    uint16_t *input_uv_table;   // 自定义 UV 量化表指针 (NULL 则使用默认)
};
```

---

## LHAL API 函数

### bflb_mjpeg_init

初始化 MJPEG 编解码器。

```c
void bflb_mjpeg_init(struct bflb_device_s *dev, const struct bflb_mjpeg_config_s *config);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `config` | `const struct bflb_mjpeg_config_s *` | MJPEG 配置结构体指针 |

---

### bflb_mjpeg_start

启动 MJPEG 摄像头压缩（与 DVP 摄像头接口联动）。

```c
void bflb_mjpeg_start(struct bflb_device_s *dev);
```

---

### bflb_mjpeg_stop

停止 MJPEG 摄像头压缩。

```c
void bflb_mjpeg_stop(struct bflb_device_s *dev);
```

---

### bflb_mjpeg_sw_run

软件触发模式：从内存缓冲区读取 YUV 数据压缩指定帧数。

```c
void bflb_mjpeg_sw_run(struct bflb_device_s *dev, uint8_t frame_count);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `frame_count` | `uint8_t` | 要压缩的帧数 |

---

### bflb_mjpeg_kick_run

启动 Kick 模式压缩（逐块压缩，无摄像头）。

```c
void bflb_mjpeg_kick_run(struct bflb_device_s *dev, uint16_t kick_count);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `kick_count` | `uint16_t` | 水平块计数 (要压缩的块数) |

---

### bflb_mjpeg_kick_stop

停止 Kick 模式压缩。

```c
void bflb_mjpeg_kick_stop(struct bflb_device_s *dev);
```

---

### bflb_mjpeg_kick

在 Kick 模式下触发一次压缩块。

```c
void bflb_mjpeg_kick(struct bflb_device_s *dev);
```

---

### bflb_mjpeg_tcint_mask

使能或禁能一帧压缩完成中断。

```c
void bflb_mjpeg_tcint_mask(struct bflb_device_s *dev, bool mask);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `mask` | `bool` | `true` = 禁能中断, `false` = 使能 |

---

### bflb_mjpeg_kickint_mask

使能或禁能 Kick 编码完成中断。（仅 BL616CL）

```c
void bflb_mjpeg_kickint_mask(struct bflb_device_s *dev, bool mask);
```

---

### bflb_mjpeg_swapint_mask

使能或禁能 Swap 交换中断。

```c
void bflb_mjpeg_swapint_mask(struct bflb_device_s *dev, bool mask);
```

---

### bflb_mjpeg_errint_mask

使能或禁能错误中断（包括所有错误类型：cam、mem、frame、idle、swap）。

```c
void bflb_mjpeg_errint_mask(struct bflb_device_s *dev, bool mask);
```

---

### bflb_mjpeg_get_intstatus

获取 MJPEG 中断状态寄存器值。

```c
uint32_t bflb_mjpeg_get_intstatus(struct bflb_device_s *dev);
```

**返回值:** 中断状态值，可使用 `MJPEG_INTSTS_*` 宏进行位判断。

---

### bflb_mjpeg_int_clear

清除 MJPEG 中断状态。

```c
void bflb_mjpeg_int_clear(struct bflb_device_s *dev, uint32_t int_clear);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `int_clear` | `uint32_t` | 清除值, 使用 `MJPEG_INTCLR_*` 宏 |

---

### bflb_mjpeg_get_frame_count

获取已压缩完成的帧数。

```c
uint8_t bflb_mjpeg_get_frame_count(struct bflb_device_s *dev);
```

**返回值:** 当前有效帧计数 (0–4)。

---

### bflb_mjpeg_pop_one_frame

丢弃一帧已压缩的帧数据，释放帧缓冲区空间。

```c
void bflb_mjpeg_pop_one_frame(struct bflb_device_s *dev);
```

---

### bflb_mjpeg_pop_swap_block

在 Swap 模式下丢弃当前块。

```c
void bflb_mjpeg_pop_swap_block(struct bflb_device_s *dev);
```

---

### bflb_mjpeg_get_swap_bit_count

获取 Swap 模式下已编码的剩余比特数。

```c
uint32_t bflb_mjpeg_get_swap_bit_count(struct bflb_device_s *dev);
```

**返回值:** 剩余比特数。

---

### bflb_mjpeg_get_frame_info

获取一帧编码后的 JPEG 帧信息。

```c
uint32_t bflb_mjpeg_get_frame_info(struct bflb_device_s *dev, uint8_t **pic);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `pic` | `uint8_t **` | 输出参数，指向 JPEG 帧数据起始地址 |

**返回值:** JPEG 帧数据长度（字节数）。

---

### bflb_mjpeg_get_swap_block_info

在 Swap 模式下获取当前交换块信息。

```c
uint8_t bflb_mjpeg_get_swap_block_info(struct bflb_device_s *dev, uint8_t *idx);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `idx` | `uint8_t *` | 输出参数，当前交换块索引 (0 或 1) |

**返回值:** `1` = 帧结束, `0` = 未结束。

---

### bflb_mjpeg_swap_is_block_full

在 Swap 模式下检查指定索引的交换块是否已满。

```c
uint8_t bflb_mjpeg_swap_is_block_full(struct bflb_device_s *dev, uint8_t idx);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `idx` | `uint8_t` | 块索引 (0 或 1) |

**返回值:** `1` = 块满, `0` = 块未满。

---

### bflb_mjpeg_calculate_quantize_table

根据质量参数计算 JPEG 量化表。

```c
void bflb_mjpeg_calculate_quantize_table(uint8_t quality, uint16_t *input_table, uint16_t *output_table);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `quality` | `uint8_t` | 图像质量 (1–100) |
| `input_table` | `uint16_t *` | 输入量化表 (64 个 uint16_t) |
| `output_table` | `uint16_t *` | 输出量化表 (64 个 uint16_t) |

---

### bflb_mjpeg_fill_quantize_table

将量化表填充到 MJPEG 硬件寄存器中。

```c
void bflb_mjpeg_fill_quantize_table(struct bflb_device_s *dev, uint16_t *input_yy, uint16_t *input_uv);
```

---

### bflb_mjpeg_fill_jpeg_header_tail

填充 JPEG 文件头到 MJPEG 硬件寄存器，并使能硬件自动添加 JPEG 文件尾(0xFFD9)。

```c
void bflb_mjpeg_fill_jpeg_header_tail(struct bflb_device_s *dev, uint8_t *header, uint32_t header_len);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `header` | `uint8_t *` | JPEG 文件头数据指针 |
| `header_len` | `uint32_t` | 文件头数据长度 |

---

### bflb_mjpeg_set_yuv420sp_cam_input

设置 YUV420SP 摄像头输入源 (指定 YY 和 UV 的 DVP 通道 ID)。

```c
void bflb_mjpeg_set_yuv420sp_cam_input(struct bflb_device_s *dev, uint8_t yy, uint8_t uv);
```

---

### bflb_mjpeg_update_input_output_buff

运行时更新 MJPEG 输入/输出缓冲区地址。

```c
void bflb_mjpeg_update_input_output_buff(struct bflb_device_s *dev, void *input_buf0, void *input_buf1, void *output_buff, size_t output_buff_size);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `input_buf0` | `void *` | 新的输入缓冲区 0 地址 (NULL 则不更新) |
| `input_buf1` | `void *` | 新的输入缓冲区 1 地址 (NULL 则不更新) |
| `output_buff` | `void *` | 新的输出缓冲区地址 (NULL 则不更新) |
| `output_buff_size` | `size_t` | 输出缓冲区大小 |

---

### bflb_mjpeg_feature_control

MJPEG 特性控制接口。

```c
int bflb_mjpeg_feature_control(struct bflb_device_s *dev, int cmd, size_t arg);
```

**参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `dev` | `struct bflb_device_s *` | 设备句柄 |
| `cmd` | `int` | 特性命令, 使用 MJPEG_CMD 宏 |
| `arg` | `size_t` | 命令参数 |

**返回值:** 成功返回 0，失败返回 `-EPERM`。`MJPEG_CMD_READ_HW_VERSION` 和 `MJPEG_CMD_READ_SW_USAGE` 返回读取到的值。

---

## 使用示例

### 示例 1: 软件模式单帧 JPEG 压缩

```c
#include "bflb_mjpeg.h"

// JPEG 文件头 (JFIF 标准)
static const uint8_t jpeg_header[] = {
    0xFF, 0xD8, 0xFF, 0xE0, 0x00, 0x10, 0x4A, 0x46,
    0x49, 0x46, 0x00, 0x01, 0x01, 0x00, 0x00, 0x01,
    0x00, 0x01, 0x00, 0x00, 0xFF, 0xDB, 0x00, 0x43,
    // ... 更多 JPEG 头数据
};

static uint8_t yuv_frame[320 * 240 * 2] __attribute__((aligned(16)));
static uint8_t jpeg_output[320 * 240 * 2 * 4] __attribute__((aligned(16)));

void mjpeg_encode_example(void)
{
    struct bflb_device_s *mjpeg;

    mjpeg = bflb_device_get_by_name("mjpeg");

    // 填充 JPEG 文件头
    bflb_mjpeg_fill_jpeg_header_tail(mjpeg, (uint8_t *)jpeg_header, sizeof(jpeg_header));

    struct bflb_mjpeg_config_s config = {
        .format = MJPEG_FORMAT_YUV422_YUYV,
        .quality = 75,
        .rows = 240,
        .resolution_x = 320,
        .resolution_y = 240,
        .input_bufaddr0 = (uint32_t)yuv_frame,
        .input_bufaddr1 = 0,
        .output_bufaddr = (uint32_t)jpeg_output,
        .output_bufsize = sizeof(jpeg_output),
        .input_yy_table = NULL,  // 使用默认量化表
        .input_uv_table = NULL,
    };

    bflb_mjpeg_init(mjpeg, &config);

    // 使能一帧完成中断
    bflb_mjpeg_tcint_mask(mjpeg, false);

    // 软件触发压缩 1 帧
    bflb_mjpeg_sw_run(mjpeg, 1);

    // 等待中断或轮询帧计数
    while (bflb_mjpeg_get_frame_count(mjpeg) == 0) {
        // 等待压缩完成 (实际应用中使用中断回调)
    }

    // 获取 JPEG 数据
    uint8_t *pic = NULL;
    uint32_t len = bflb_mjpeg_get_frame_info(mjpeg, &pic);

    // pic 指向有效的 JPEG 数据，len 为字节数
    // 可以发送或保存 pic[0..len-1]

    // 丢弃已处理的帧
    bflb_mjpeg_pop_one_frame(mjpeg);
}
```

### 示例 2: 摄像头模式实时 MJPEG 编码

```c
void mjpeg_camera_example(void)
{
    struct bflb_device_s *mjpeg;

    mjpeg = bflb_device_get_by_name("mjpeg");

    // 填充 JPEG 文件头
    bflb_mjpeg_fill_jpeg_header_tail(mjpeg, (uint8_t *)jpeg_header, sizeof(jpeg_header));

    struct bflb_mjpeg_config_s config = {
        .format = MJPEG_FORMAT_YUV420SP_NV12,
        .quality = 80,
        .rows = 480,
        .resolution_x = 640,
        .resolution_y = 480,
        .input_bufaddr0 = (uint32_t)yy_buffer,
        .input_bufaddr1 = (uint32_t)uv_buffer,
        .output_bufaddr = (uint32_t)jpeg_output,
        .output_bufsize = 640 * 480 * 2 * 4,
        .input_yy_table = NULL,
        .input_uv_table = NULL,
    };

    bflb_mjpeg_init(mjpeg, &config);

    // 设置摄像头 YUV420SP 输入源
    bflb_mjpeg_set_yuv420sp_cam_input(mjpeg, 0, 0);

    // 使能中断
    bflb_mjpeg_tcint_mask(mjpeg, false);

    // 启动摄像头模式压缩
    bflb_mjpeg_start(mjpeg);

    // 在主循环中处理帧
    while (1) {
        if (bflb_mjpeg_get_frame_count(mjpeg) > 0) {
            uint8_t *pic = NULL;
            uint32_t len = bflb_mjpeg_get_frame_info(mjpeg, &pic);

            // 发送或保存 JPEG 帧
            // send_jpeg_frame(pic, len);

            bflb_mjpeg_pop_one_frame(mjpeg);
        }
    }
}
```

### 示例 3: 自定义量化表 + Swap 模式

```c
static uint16_t my_quant_y[64] = {
    16, 11, 10, 16, 24, 40, 51, 61,
    12, 12, 14, 19, 26, 58, 60, 55,
    14, 13, 16, 24, 40, 57, 69, 56,
    14, 17, 22, 29, 51, 87, 80, 62,
    18, 22, 37, 56, 68, 109, 103, 77,
    24, 35, 55, 64, 81, 104, 113, 92,
    49, 64, 78, 87, 103, 121, 120, 101,
    72, 92, 95, 98, 112, 100, 103, 99
};

static uint16_t my_quant_uv[64] = {
    17, 18, 24, 47, 99, 99, 99, 99,
    // ... UV 量化表 64 项
};

void mjpeg_custom_quality_example(void)
{
    struct bflb_device_s *mjpeg;

    mjpeg = bflb_device_get_by_name("mjpeg");

    // 计算量化表
    uint16_t quant_y[64], quant_uv[64];
    bflb_mjpeg_calculate_quantize_table(60, my_quant_y, quant_y);
    bflb_mjpeg_calculate_quantize_table(60, my_quant_uv, quant_uv);

    struct bflb_mjpeg_config_s config = {
        .format = MJPEG_FORMAT_YUV422_YUYV,
        .quality = 60,
        .rows = 480,
        .resolution_x = 640,
        .resolution_y = 480,
        .input_bufaddr0 = (uint32_t)yuv_frame,
        .input_bufaddr1 = 0,
        .output_bufaddr = (uint32_t)jpeg_output,
        .output_bufsize = sizeof(jpeg_output),
        .input_yy_table = my_quant_y,  // 使用自定义量化表
        .input_uv_table = my_quant_uv,
    };

    bflb_mjpeg_init(mjpeg, &config);

    // 使能 Swap 模式 (双块交换输出)
    bflb_mjpeg_feature_control(mjpeg, MJPEG_CMD_SWAP_ENABLE, 1);

    // 使能 Swap 中断
    bflb_mjpeg_swapint_mask(mjpeg, false);

    // 启动压缩
    bflb_mjpeg_sw_run(mjpeg, 1);

    // 轮询 Swap 块状态
    uint8_t idx;
    while (!bflb_mjpeg_get_swap_block_info(mjpeg, &idx)) {
        if (bflb_mjpeg_swap_is_block_full(mjpeg, idx)) {
            uint32_t bits = bflb_mjpeg_get_swap_bit_count(mjpeg);
            // 处理满的 swap 块
            bflb_mjpeg_pop_swap_block(mjpeg);
        }
    }
}
```

### 示例 4: 中断处理模式

```c
void mjpeg_interrupt_example(void)
{
    struct bflb_device_s *mjpeg;

    mjpeg = bflb_device_get_by_name("mjpeg");

    // ... 初始化 MJPEG 配置 ...

    // 使能一帧完成中断和错误中断
    bflb_mjpeg_tcint_mask(mjpeg, false);
    bflb_mjpeg_errint_mask(mjpeg, false);

    // 启动摄像头模式
    bflb_mjpeg_start(mjpeg);
}

// 中断服务例程
void mjpeg_isr_handler(void)
{
    struct bflb_device_s *mjpeg = bflb_device_get_by_name("mjpeg");
    uint32_t status = bflb_mjpeg_get_intstatus(mjpeg);

    if (status & MJPEG_INTSTS_ONE_FRAME) {
        // 一帧完成
        uint8_t *pic = NULL;
        uint32_t len = bflb_mjpeg_get_frame_info(mjpeg, &pic);

        // 处理 JPEG 帧...

        bflb_mjpeg_pop_one_frame(mjpeg);

        // 清除中断
        bflb_mjpeg_int_clear(mjpeg, MJPEG_INTCLR_ONE_FRAME);
    }

    if (status & MJPEG_INTSTS_SWAP) {
        // 交换中断处理
        bflb_mjpeg_int_clear(mjpeg, MJPEG_INTCLR_SWAP);
    }
}
```

---

## 寄存器信息

MJPEG 模块寄存器基地址通过 `dev->reg_base` 获取。

### 核心控制寄存器

#### MJPEG_CONTROL_1 (偏移 0x00)

| 位 | 字段 | 说明 |
|----|------|------|
| 0 | `MJPEG_ENABLE` | MJPEG 使能 (1=使能) |
| 1 | `MJPEG_BIT_ORDER` | JPEG 比特顺序 |
| 2 | `ORDER_U_EVEN` | U 分量排序 (1=UV, 0=VU) |
| 3 | `HW_MODE_SWEN` | 硬件模式软件使能 |
| 4 | `LAST_HF_WBLK_DMY` | 最后半宽块填充 |
| 5 | `LAST_HF_HBLK_DMY` | 最后半高块填充 |
| 7 | `READ_FWRAP` | 帧读取环绕使能 |
| 8–10 | `W_XLEN` | AXI 写突发长度 |
| 12–13 | `YUV_MODE` | YUV 模式 (0=420, 1=Gray, 2=422SP, 3=422) |
| 14 | `KICK_DONE_STS_EN` | Kick 完成状态使能 (仅 BL616CL) |
| 24–29 | `MJPEG_HW_FRAME` | 硬件帧计数 |
| 30 | `KICK_UPDATE_ADDR` | Kick 更新地址 (仅 BL616CL) |

#### MJPEG_CONTROL_2 (偏移 0x04)

| 位 | 字段 | 说明 |
|----|------|------|
| 0–4 | `SW_FRAME` | 软件触发帧数 (0–31) |
| 5 | `INT_KICK_CLR` | Kick 中断清除 (仅 BL616CL) |
| 6 | `SW_KICK` | 软件 Kick 触发 |
| 7 | `SW_KICK_MODE` | Kick 模式使能 |
| 8 | `MJPEG_SW_MODE` | 软件模式使能 |
| 9 | `MJPEG_SW_RUN` | 软件运行 |
| 10–12 | `YY_DVP2AXI_SEL` | YY 分量 DVP→AXI 通道选择 |
| 13–15 | `UV_DVP2AXI_SEL` | UV 分量 DVP→AXI 通道选择 |
| 16–31 | `MJPEG_WAIT_CYCLE` | 等待周期(默认 0x100) |

#### MJPEG_CONTROL_3 (偏移 0x1C)

| 位 | 字段 | 说明 |
|----|------|------|
| 0 | `INT_NORMAL_EN` | 正常中断使能 |
| 1 | `INT_CAM_EN` | 摄像头错误中断使能 |
| 2 | `INT_MEM_EN` | 内存错误中断使能 |
| 3 | `INT_FRAME_EN` | 帧错误中断使能 |
| 4 | `STS_NORMAL_INT` | 正常中断状态 |
| 5 | `STS_CAM_INT` | 摄像头错误中断状态 |
| 6 | `STS_MEM_INT` | 内存错误中断状态 |
| 7 | `STS_FRAME_INT` | 帧错误中断状态 |
| 8–15 | 状态标志 | IDLE/FUNC/WAIT 等状态 |
| 16–20 | `FRAME_CNT_TRGR_INT` | 帧触发中断阈值 |
| 21 | `INT_IDLE_EN` | 空闲中断使能 |
| 22 | `STS_IDLE_INT` | 空闲中断状态 |
| 23 | `STS_KICK_INT` | Kick 中断状态 (仅 BL616CL) |
| 24–28 | `FRAME_VALID_CNT` | 有效帧计数 (0–4) |
| 29 | `INT_SWAP_EN` | Swap 中断使能 |
| 30 | `STS_SWAP_INT` | Swap 中断状态 |
| 31 | `INT_KICK_EN` | Kick 中断使能 (仅 BL616CL) |

### 缓冲区地址寄存器

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `MJPEG_YY_FRAME_ADDR` | `0x08` | YY 帧缓冲区地址 |
| `MJPEG_UV_FRAME_ADDR` | `0x0C` | UV 帧缓冲区地址 |
| `MJPEG_JPEG_FRAME_ADDR` | `0x14` | JPEG 输出缓冲区地址 |
| `MJPEG_JPEG_STORE_MEMORY` | `0x18` | JPEG 存储空间大小 (burst 计数 = size/128) |

### 帧信息寄存器

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `MJPEG_START_ADDR0–3` | `0x80/88/90/98` | 帧 0–3 的起始地址 |
| `MJPEG_BIT_CNT0–3` | `0x84/8C/94/9C` | 帧 0–3 的比特计数 |

### 帧尺寸和头部

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `MJPEG_FRAME_SIZE` | `0x24` | 帧宽度块数 (bits 0–11) + 帧高度块数 (bits 16–27) |
| `MJPEG_HEADER_BYTE` | `0x28` | JPEG 头部长度 + YUV 分量排布顺序 + 尾部使能 |

### YUV 内存配置

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `MJPEG_YUV_MEM` | `0x10` | YY hblk 计数 (bits 0–12) + UV hblk 计数 (bits 16–28) |
| `MJPEG_YUV_MEM_SW` | `0x38` | 软件模式 Kick hblk 计数 |

### Swap 模式寄存器

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `MJPEG_SWAP_MODE` | `0x30` | Swap 模式配置和状态 |
| `MJPEG_SWAP_BIT_CNT` | `0x34` | Swap 帧结束比特计数 |

### 量化编码寄存器

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `MJPEG_Q_ENC` | `0x100` | 量化编码控制 (bit 24=Q_SRAM_SW) |
| `MJPEG_Q_PARAM_00` | `0x400–0x47C` | Y 量化表 (8×4 个 32-bit 寄存器) |
| `MJPEG_Q_PARAM_40` | `0x480–0x4FC` | UV 量化表 (8×4 个 32-bit 寄存器) |

### 其他寄存器

| 寄存器 | 偏移 | 说明 |
|--------|------|------|
| `MJPEG_FRAME_FIFO_POP` | `0x20` | 帧/中断弹出和清除寄存器 |
| `MJPEG_KICK_DONE_DELAY` | `0x104` | Kick 完成延迟 (仅 BL616CL) |
| `MJPEG_FRAME_ID_10` | `0x110` | 帧 0 和帧 1 的 ID |
| `MJPEG_FRAME_ID_32` | `0x114` | 帧 2 和帧 3 的 ID |
| `MJPEG_DEBUG` | `0x1F0` | 调试控制 |
| `MJPEG_SW_USAGE` | `0x1F8` | 软件使用标志 (仅 BL616CL) |
| `MJPEG_HW_VERSION` | `0x1F8` | 硬件版本 (仅 BL616CL) |
| `MJPEG_DUMMY_REG` | `0x1FC` | 虚拟寄存器 |

> **注意：** 量化表寄存器布局为 8 列 × 4 行，每个寄存器包含两个 uint16_t 值 (低 16 位 + 高 16 位)。
