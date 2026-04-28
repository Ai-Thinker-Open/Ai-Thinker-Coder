# BL616/BL618 AI 硬件加速技术文档

## 概述

BL616 和 BL618 是博流半导体（Bouffalo Lab）推出的高性能 Wi-Fi 6 + Bluetooth LE 5.x  combo 芯片，集成 DNN（Deep Neural Network）硬件加速器，专门针对边缘 AI 推理场景进行优化。该芯片系列支持多种 AI 应用，包括图像分类、目标检测、语音唤醒（KWS）和手势识别等。

BL618 作为 BL616 的多媒体增强版本，在 AI 推理能力上更为强大，提供了更大的内存带宽和更丰富的外设接口，特别适合需要处理图像和语音的 AIoT 应用场景。

## 1. 硬件架构

### 1.1 DNN/CNN 硬件加速器

BL616/BL618 集成了专门的 CNN（卷积神经网络）硬件加速器模块，该模块在 SDK 中通过多媒体时钟管理系统（MM GLB）进行控制和管理。从寄存器定义可以观察到以下关键特性：

- **CNN 时钟使能**：`MM_GLB_REG_CNN_CLK_DIV_EN` 用于启用 CNN 时钟
- **CNN 时钟选择**：`MM_GLB_REG_CNN_CLK_SEL` 支持 2 位时钟源选择
- **CNN 时钟分频**：`MM_GLB_REG_CNN_CLK_DIV` 提供 3 位分频系数
- **CNN 软件复位**：`MM_GLB_SWRST_CNN` 用于模块复位控制

CNN 加速器采用独立时钟域设计，可以独立于 CPU 主频运行，这种设计使得 AI 推理任务可以在不影响系统响应的前提下高效执行。

### 1.2 处理器配置

BL616/BL618 内置 RISC-V E907 处理器核心，具备以下计算能力：

- **FPU（浮点单元）**：支持单精度和双精度浮点运算，加速 AI 模型中的激活函数计算
- **DSP（数字信号处理器）**：支持 SIMD（单指令多数据）操作，可加速卷积和矩阵运算
- **紧耦合内存**：用于关键算法的快速数据访问

### 1.3 内存系统

#### BL618 多媒体增强特性

BL618 在内存配置上进行了显著增强，这是支持 AI 推理的关键：

| 内存类型 | 容量 | 用途 |
|---------|------|------|
| PSRAM | 4MB | 模型权重、中间结果缓存、大图像帧缓冲 |
| SRAM | 532KB | 运行时数据、堆栈、实时处理数据 |

**PSRAM 地址映射**：

- BL618DG X8 PSRAM：0x88000000
- BL616 X8 PSRAM：0xA8000000
- BL616CL X8 PSRAM：0x88000000
- UHS PSRAM：0x50000000

PSRAM 的 4MB 大容量设计使得可以同时存储大型 AI 模型权重和多个图像帧的中间结果，这对于实时图像处理至关重要。相比之下，532KB 的 SRAM 则用于需要快速访问的实时数据和系统运行时状态。

## 2. 多媒体接口

### 2.1 Camera/DVP 接口

BL616/BL618 支持 DVP（Digital Video Port）摄像头接口，这是连接图像传感器获取实时视频流的关键接口。SDK 中定义的支持的图像传感器包括：

| 型号 | 接口类型 | 支持芯片 |
|------|----------|----------|
| gc2053 | DVP/CSI | BL702/BL616/BL808 |
| gc0308 | DVP | BL702/BL616/BL808 |
| gc0328 | DVP | BL702/BL616/BL808 |
| ov2640 | DVP | BL702/BL616/BL808 |
| bf2013 | DVP | BL702/BL616/BL808 |
| bf20a6 | DVP/SPI | 多种配置 |

DVP 接口支持多种数据格式和分辨率配置，可通过 `bflb_cam.h` 中定义的 `CAM_INPUT_SOURCE_DVP` 进行源选择。摄像头接口与 CNN 加速器的协同工作构成了完整的视觉 AI 处理流水线。

### 2.2 显示输出接口

BL618 支持 DPI（Display Parallel Interface）显示接口，可用于输出处理后的图像结果。DVP 到显示的数据传输通过 `bflb_dpi.c` 驱动程序实现，支持多种像素格式配置。

## 3. AI 应用场景

### 3.1 图像识别（CNN）

卷积神经网络（CNN）硬件加速器是 BL616/BL618 AI 能力的核心。该加速器专门优化了卷积运算，这是 CNN 中计算量最大的操作。典型的应用场景包括：

- **图像分类**：将输入图像分类到预定义的类别，如动物、植物、产品等
- **目标检测**：定位图像中的多个目标物体，并标记出边界框和类别
- **人脸检测**：在图像中检测人脸位置，可用于考勤系统、智能门锁等

CNN 加速器支持多种卷积优化策略，包括滑动窗口优化、Winograd 算法加速等，可以在保持精度的前提下显著提升推理速度。

### 3.2 语音唤醒（KWS）

关键词识别（Keyword Spotting）是语音 AI 的基础应用。BL616/BL618 通过音频采集接口配合软件算法实现低功耗语音唤醒功能。典型的唤醒词检测流程包括：

1. **音频采集**：通过 I2S 或 PDM 接口采集麦克风数据
2. **特征提取**：将时域音频信号转换为频域特征（如 MFCC）
3. **模型推理**：使用小型神经网络判断是否包含唤醒词
4. **结果输出**：检测到唤醒词后触发相应动作

KWS 系统的设计重点在于低功耗和实时性，BL618 的 DSP 扩展指令集可以加速 MFCC 计算，而专用的 CNN 加速器则可运行轻量级分类网络。

### 3.3 手势识别

手势识别结合了视觉感知和时序分析，通常采用以下技术方案：

- **空间特征提取**：使用 CNN 从单帧图像中提取手势的空间特征
- **时序建模**：使用 RNN/LSTM 或 1D CNN 处理多帧序列的时间依赖关系
- **分类输出**：最终判断手势类别

BL618 的 4MB PSRAM 为手势识别提供了充足的缓冲区，可以缓存多帧图像数据用于时序分析，而 532KB SRAM 则保证实时处理所需的低延迟数据访问。

## 4. API 参考

### 4.1 DNN 初始化与模型加载

DNN（Deep Neural Network）模块是 CNN 加速器的软件抽象层，负责模型加载和推理调度。以下是典型的 DNN 使用流程：

```c
#include "bflb_dnn.h"

// DNN 上下文句柄
bflb_dnn_context_t dnn_ctx;

// DNN 配置参数
bflb_dnn_config_t dnn_config = {
    .priority = 5,              // 任务优先级
    .timeout_ms = 1000,         // 推理超时时间
    .psram_buffer = 1,          // 使用 PSRAM 作为数据缓冲
};

// 初始化 DNN 模块
int ret = bflb_dnn_init(&dnn_ctx, &dnn_config);
if (ret < 0) {
    printf("DNN init failed: %d\r\n", ret);
    return ret;
}

// 加载模型数据（模型数据通常存储在 Flash 或文件系统）
// 模型格式可能为供应商特定的二进制格式
ret = bflb_dnn_load_model(&dnn_ctx, model_data, model_size);
if (ret < 0) {
    printf("Model load failed: %d\r\n", ret);
    bflb_dnn_deinit(&dnn_ctx);
    return ret;
}

printf("DNN model loaded successfully\r\n");
```

### 4.2 推理执行

模型加载完成后，可以对输入数据进行推理。输入数据通常来自摄像头或其他传感器：

```c
// 准备输入数据（假设来自摄像头的 RGB 图像）
// 图像数据通常放置在 PSRAM 中以节省 SRAM 空间
uint8_t *image_input = (uint8_t *)psram_malloc(224 * 224 * 3);
if (image_input == NULL) {
    printf("Failed to allocate PSRAM for input\r\n");
    return -1;
}

// 从摄像头获取的图像数据填充到输入缓冲区
// fill_image_data_from_camera(image_input, 224, 224);

// 准备输出缓冲区
float output_buffer[NUM_CLASSES];

// 创建输入输出张量描述
bflb_dnn_tensor_t input_tensor = {
    .data = image_input,
    .shape = {1, 224, 224, 3},  // NCHW 格式
    .dtype = BFLB_DNN_DTYPE_UINT8,
};

bflb_dnn_tensor_t output_tensor = {
    .data = output_buffer,
    .shape = {1, NUM_CLASSES},
    .dtype = BFLB_DNN_DTYPE_FLOAT32,
};

// 执行推理
bflb_dnn_inference_t inference = {
    .input = &input_tensor,
    .output = &output_tensor,
};

ret = bflb_dnn_run_inference(&dnn_ctx, &inference);
if (ret < 0) {
    printf("Inference failed: %d\r\n", ret);
} else {
    // 解析输出结果
    int predicted_class = argmax(output_buffer, NUM_CLASSES);
    printf("Predicted class: %d\r\n", predicted_class);
}

// 清理资源
psram_free(image_input);
```

### 4.3 图像帧输入处理

对于实时视频流处理，需要连续处理多帧图像。以下是典型的帧处理流程：

```c
#include "bflb_cam.h"
#include "bflb_dnn.h"

// 假设已初始化摄像头和 DNN
bflb_cam_device_t cam;
bflb_dnn_context_t dnn_ctx;

// 分配帧缓冲区（使用 PSRAM 存储大尺寸图像）
#define FRAME_WIDTH 320
#define FRAME_HEIGHT 240
#define FRAME_BUFFER_COUNT 2

uint8_t *frame_buffers[FRAME_BUFFER_COUNT];

for (int i = 0; i < FRAME_BUFFER_COUNT; i++) {
    frame_buffers[i] = psram_malloc(FRAME_WIDTH * FRAME_HEIGHT * 2);  // RGB565
}

// 分配预处理缓冲区
uint8_t *preprocess_buffer = psram_malloc(224 * 224 * 3);

// 图像预处理函数
void preprocess_image(uint8_t *src, uint8_t *dst, int src_w, int src_h, int dst_size) {
    // 1. 缩放图像到模型输入尺寸
    // 2. 颜色空间转换（如需要）
    // 3. 归一化处理
    // 这里使用简单的最近邻缩放作为示例
    scale_image_nearest_neighbor(src, dst, src_w, src_h, dst_size, dst_size);
    normalize_image(dst, dst_size * dst_size * 3);
}

// 主循环处理视频帧
while (1) {
    // 等待摄像头帧中断（简化表示）
    bflb_cam_frame_t *frame = bflb_cam_get_frame(&cam, 1000);
    if (frame == NULL) {
        continue;
    }
    
    // 图像预处理（缩放、归一化）
    preprocess_image(frame->data, preprocess_buffer, 
                     FRAME_WIDTH, FRAME_HEIGHT, 224);
    
    // 执行 AI 推理
    bflb_dnn_tensor_t input_tensor = {
        .data = preprocess_buffer,
        .shape = {1, 224, 224, 3},
        .dtype = BFLB_DNN_DTYPE_UINT8,
    };
    
    float output[NUM_CLASSES];
    bflb_dnn_tensor_t output_tensor = {
        .data = output,
        .shape = {1, NUM_CLASSES},
        .dtype = BFLB_DNN_DTYPE_FLOAT32,
    };
    
    bflb_dnn_inference_t inference = {
        .input = &input_tensor,
        .output = &output_tensor,
    };
    
    int ret = bflb_dnn_run_inference(&dnn_ctx, &inference);
    if (ret == 0) {
        // 处理推理结果
        handle_inference_result(output);
    }
    
    // 释放帧缓冲
    bflb_cam_release_frame(&cam, frame);
}

// 清理
for (int i = 0; i < FRAME_BUFFER_COUNT; i++) {
    psram_free(frame_buffers[i]);
}
psram_free(preprocess_buffer);
```

### 4.4 KWS 关键词识别

语音唤醒通常采用独立的软件算法实现，配合音频采集外设工作：

```c
#include "bflb_kws.h"
#include "bflb_i2s.h"

// KWS 配置
bflb_kws_config_t kws_config = {
    .sample_rate = 16000,
    .frame_size = 512,
    .hop_size = 160,
    .num_mfcc = 13,
    .detection_threshold = 0.75,
    .model_type = KWS_MODEL_MLP,
};

// 初始化 KWS
bflb_kws_context_t kws_ctx;
int ret = bflb_kws_init(&kws_ctx, &kws_config);
if (ret < 0) {
    printf("KWS init failed: %d\r\n", ret);
    return ret;
}

// 加载关键词检测模型
ret = bflb_kws_load_model(&kws_ctx, kws_model_data, kws_model_size);
if (ret < 0) {
    printf("KWS model load failed: %d\r\n", ret);
    return ret;
}

// 音频缓冲区
int16_t audio_buffer[512];

// KWS 检测循环
while (1) {
    // 从 I2S 获取音频数据
    ret = bflb_i2s_read(&i2s_device, audio_buffer, 512);
    if (ret < 0) {
        continue;
    }
    
    // 运行 KWS 检测
    float score = 0.0f;
    int detected = bflb_kws_detect(&kws_ctx, audio_buffer, 512, &score);
    
    if (detected) {
        printf("Keyword detected! score=%.2f\r\n", score);
        // 触发后续处理，如启动语音识别或命令处理
    }
}

// 清理
bflb_kws_deinit(&kws_ctx);
```

## 5. PSRAM 在 AI 推理中的角色

PSRAM（Pseudo Static RAM）是 BL618 支持的外部存储芯片，在 AI 推理中扮演着至关重要的角色。

### 5.1 模型权重存储

现代深度学习模型的参数量从数百万到数亿不等。以常见的 MobileNetV2 为例，其模型大小约为 14MB，远超芯片内置 SRAM 容量。4MB PSRAM 虽然不能存储完整的大型模型，但可以：

- **分级存储策略**：将模型分层，常用层权重保留在 PSRAM，不常用层从 Flash 按需加载
- **模型分片**：将大模型分割为多个小片段，按执行顺序加载到 PSRAM
- **混合推理**：部分层在 PSRAM，部分在 Flash，通过 DMA 高效传输

### 5.2 中间结果缓存

CNN 推理过程中，每一层的输出特征图（Feature Map）都需要临时存储：

- **特征图缓存**：卷积层的输出需要保存作为下一层的输入，PSRAM 的大容量可以缓存多层特征图
- **多任务共享**：当运行多个 AI 任务时（如同时做人脸检测和手势识别），PSRAM 可以分别保存各自的中途结果
- **视频帧缓冲**：对于手势识别等时序应用，需要缓存多帧图像，PSRAM 的空间可以容纳数十帧QVGA图像

### 5.3 图像帧缓冲

在图像处理应用中，PSRAM 用于存储来自摄像头的原始图像：

```c
// JPEG 解码示例 - 使用 PSRAM 作为解码缓冲区
ATTR_NOINIT_PSRAM_SECTION __ALIGNED(64) uint8_t jpeg_buff[32 * 1024];

// 显示缓冲区
ATTR_NOINIT_PSRAM_SECTION __ALIGNED(64) uint8_t bpp_buff[1][320 * 240 * N_BPP];
```

PSRAM 的 64 字节对齐特性有利于 DMA 传输，可以配合摄像头和显示外设实现零拷贝图像处理。

## 6. Camera 与 Display 协同工作

BL618 的多媒体处理流水线支持 Camera 输入、AI 推理、Display 输出的完整链路。

### 6.1 数据流架构

```
┌─────────┐    DVP     ┌──────────┐   DMA/PSRAM   ┌─────────┐   AXI     ┌──────────┐
│ Camera  │ ────────▶  │  Camera  │ ───────────▶ │  PSRAM  │ ────────▶ │   CNN    │
│ Sensor  │            │  Interface│              │ Buffer  │           │ Accelerator│
└─────────┘            └──────────┘              └─────────┘           └──────────┘
                                                                           │
                                                                           ▼
┌─────────┐    DVP     ┌──────────┐   DMA/PSRAM   ┌─────────┐           ┌──────────┐
│ Display │ ◀────────  │   DPI    │ ◀─────────── │  OSD    │ ◀──────── │  Results │
│  Panel  │            │ Interface │              │ Buffer  │           │  Process │
└─────────┘            └──────────┘              └─────────┘           └──────────┘
```

### 6.2 完整应用示例

以下代码展示了一个完整的人脸检测应用流程：

```c
#include "bflb_cam.h"
#include "bflb_dnn.h"
#include "bflb_dpi.h"
#include "bl616_psram.h"

// 系统初始化
void ai_vision_system_init(void) {
    // 1. 初始化 PSRAM
    bl_psram_init();
    
    // 2. 初始化摄像头接口
    bflb_cam_config_t cam_config = {
        .input_source = CAM_INPUT_SOURCE_DVP,
        .width = 320,
        .height = 240,
        .format = CAM_FORMAT_YUV422_YUYV,
    };
    bflb_cam_init(&cam, &cam_config);
    
    // 3. 初始化显示接口
    bflb_dpi_config_t dpi_config = {
        .width = 320,
        .height = 240,
        .format = DPI_FORMAT_RGB565,
    };
    bflb_dpi_init(&dpi, &dpi_config);
    
    // 4. 初始化 DNN 加速器
    bflb_dnn_config_t dnn_config = {
        .priority = 5,
        .timeout_ms = 2000,
        .psram_buffer = 1,
    };
    bflb_dnn_init(&dnn_ctx, &dnn_config);
    bflb_dnn_load_model(&dnn_ctx, face_detection_model, model_size);
}

// 主循环
void ai_vision_task(void *params) {
    uint8_t *camera_frame = psram_malloc(320 * 240 * 2);
    uint8_t *preprocessed = psram_malloc(224 * 224 * 3);
    uint8_t *osd_overlay = psram_malloc(320 * 240 * 2);
    
    while (1) {
        // 获取摄像头帧
        bflb_cam_frame_t *frame = bflb_cam_get_frame(&cam, 1000);
        if (!frame) continue;
        
        // 复制到 PSRAM 缓冲区
        memcpy(camera_frame, frame->data, frame->size);
        bflb_cam_release_frame(&cam, frame);
        
        // 图像预处理
        convert_yuv_to_rgb(camera_frame, preprocessed, 320, 240);
        resize_and_normalize(preprocessed, 224, 224);
        
        // AI 推理 - 人脸检测
        float boxes[10][4];  // 最多检测 10 个人脸
        int face_count = run_face_detection(&dnn_ctx, preprocessed, boxes);
        
        // 生成 OSD 叠加层（绘制检测框）
        memset(osd_overlay, 0, 320 * 240 * 2);
        for (int i = 0; i < face_count; i++) {
            draw_rectangle(osd_overlay, boxes[i], 320, 240);
        }
        
        // 叠加 OSD 并显示
        bflb_dpi_send_frame(&dpi, osd_overlay, 320 * 240 * 2);
    }
    
    psram_free(camera_frame);
    psram_free(preprocessed);
    psram_free(osd_overlay);
}
```

## 7. 性能优化建议

### 7.1 内存访问优化

- **PSRAM vs SRAM 选择**：频繁访问的数据使用 SRAM，大缓冲区使用 PSRAM
- **对齐访问**：确保数据结构按照 64 字节对齐，以获得最佳 DMA 性能
- **缓存优化**：善用处理器缓存，减少 PSRAM 访问延迟

### 7.2 推理优化

- **模型量化**：将 float32 模型量化到 int8，可减少 75% 内存占用并提升推理速度
- **层融合**：将相邻的卷积、BatchNorm、ReLU 层融合为单一操作
- **输入尺寸优化**：选择与模型训练时最接近的输入尺寸，避免额外的缩放计算

### 7.3 功耗优化

- **动态频率调节**：根据推理复杂度动态调整 CNN 时钟频率
- **中断驱动**：使用中断而非轮询处理摄像头帧和推理完成事件
- **休眠策略**：无任务时让 CPU 进入低功耗模式，仅保留 CNN 加速器监听

## 8. 注意事项

### 8.1 开发限制

- **模型大小**：受限于 PSRAM 容量，单一模型通常不超过 2MB（量化后）
- **实时性**：复杂模型可能导致推理时间超过 100ms，需根据应用场景权衡
- **内存布局**：PSRAM 地址映射因芯片型号不同而异，务必参考对应芯片的数据手册

### 8.2 调试建议

- **先验证硬件**：确保 Camera、DVP、PSRAM 等外设正常工作后再进行 AI 开发
- **逐级测试**：先测试数据采集，再测试预处理，最后集成 AI 推理
- **资源监控**：定期检查内存使用情况，避免内存泄漏导致系统崩溃

### 8.3 常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| 推理返回错误 | 模型格式不匹配 | 确认模型与 DNN 加速器版本兼容 |
| 图像显示异常 | 帧缓冲格式错误 | 检查 Camera 和 DPI 的像素格式配置 |
| 内存分配失败 | PSRAM 未初始化 | 调用 bl_psram_init() 初始化 PSRAM |
| 推理结果不正确 | 输入预处理不一致 | 确保预处理与模型训练时的数据处理一致 |

## 参考

- [Bouffalo SDK 官方文档](https://github.com/bouffalo-lab/bouffalo_sdk)
- [BL618 参考手册](./bl618.md)
- [PSRAM 驱动使用指南](./psram.md)
- [Camera 接口开发指南](./camera.md)
- [DNN 模型转换工具文档](./model_converter.md)
- Bouffalo Lab 官方 SDK 仓库：https://github.com/bouffalo-lab/bouffalo_sdk
- BL616/BL618 芯片系列 Datasheet
