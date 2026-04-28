# SDIO/SDH Driver Documentation

**Hardware Base:** `SDH_BASE = 0x2000C800`

**芯片支持:** BL616, BL616CL, BL618DG

---

## 1. 架构概述

| 模块 | 说明 |
|------|------|
| **SDH** | Secure Digital Host - 底层SD/MMC控制器，支持DMA/ADMA |
| **SDIO2** | SDIO 2.0协议层 (向后兼容) |
| **SDIO3** | SDIO 3.0协议层 (高性能，支持SDR104/SDR50/DDR50) |

---

## 2. SDH (Secure Digital Host)

### 2.1 关键特性

- **数据传输方向:** `SDH_TRANSFER_DIR_WR` (发送) / `SDH_TRANSFER_DIR_RD` (接收)
- **数据总线宽度:** 1位 / 4位 / 8位
- **Endian模式:** Little / Half-Word Big / Big
- **DMA类型:** SDMA / ADMA1 / ADMA2
- **支持SDIO中断**

### 2.2 状态标志

```c
#define SDH_NORMAL_STA_CARD_INSERT    (1 << 6)  // 卡插入
#define SDH_NORMAL_STA_CARD_REMOVE    (1 << 7)  // 卡移除
#define SDH_NORMAL_STA_CMD_COMP       (1 << 0)  // 命令完成
#define SDH_NORMAL_STA_TRAN_COMP      (1 << 1)  // 传输完成
#define SDH_NORMAL_STA_DMA_INT        (1 << 3)  // DMA中断
#define SDH_NORMAL_STA_BUFF_WR_RDY    (1 << 4)  // 缓冲区写就绪
#define SDH_NORMAL_STA_BUFF_RD_RDY    (1 << 5)  // 缓冲区读就绪
```

### 2.3 错误状态

```c
#define SDH_ERROR_STA_CMD_TIMEOUT     (1 << 16)  // 命令超时
#define SDH_ERROR_STA_CMD_CRC_ERR     (1 << 17)  // 命令CRC错误
#define SDH_ERROR_STA_DATA_TIMEOUT    (1 << 20)  // 数据超时
#define SDH_ERROR_STA_DATA_CRC_ERR    (1 << 21)  // 数据CRC错误
#define SDH_ERROR_STA_ADMA_ERR        (1 << 25)  // ADMA错误
```

---

## 3. 数据结构

### 3.1 SDH配置

```c
struct bflb_sdh_config_s {
    uint8_t dma_fifo_th;  // FIFO阈值: 64/128/192/256
    uint8_t dma_burst;   // 突发长度: 32/64/128/256
    uint8_t power_vol;   // 电源电压
};
```

### 3.2 命令配置

```c
struct bflb_sdh_cmd_cfg_s {
    uint8_t index;      // 命令索引 (0-63)
    uint8_t cmd_type;   // 命令类型: NORMAL/SUSPEND/RESUME/ABORT
    uint8_t resp_type;  // 响应类型: NONE/R1/R1b/R2/R3/R4/R5/R5b/R6/R7
    uint32_t argument;  // 命令参数
    uint32_t resp[4];   // 响应数据
};
```

### 3.3 数据配置

```c
struct bflb_sdh_data_cfg_s {
    uint8_t data_dir;      // SDH_TRANSFER_DIR_RD 或 SDH_TRANSFER_DIR_WR
    uint8_t data_type;     // NORMAL/TUNING/BOOT/BOOT_CONTINU
    uint8_t auto_cmd_mode; // AUTO_CMD_DISABLE/CMD12/CMD23
    uint16_t block_size;   // 块大小
    uint16_t block_count;  // 块数量
    
    // ADMA2配置
    bool adma2_hw_desc_raw_mode;
    struct bflb_sdh_data_tranfer_s *adma_tranfer;
    uint32_t adma_tranfer_cnt;
    struct bflb_sdh_adma2_hw_desc_s *adma2_hw_desc;
    uint32_t adma2_hw_desc_cnt;
};
```

---

## 4. API参考

### 4.1 初始化

```c
int bflb_sdh_init(struct bflb_device_s *dev, struct bflb_sdh_config_s *cfg);
```

**示例:**
```c
struct bflb_device_s *sdh;
struct bflb_sdh_config_s sdh_cfg = {
    .dma_fifo_th = SDH_DMA_FIFO_THRESHOLD_256,
    .dma_burst = SDH_DMA_BURST_256,
    .power_vol = 0,
};

sdh = bflb_device_get_by_name("sdh");
if (sdh) {
    bflb_sdh_init(sdh, &sdh_cfg);
}
```

### 4.2 卡插入检测

```c
// 启用卡插入状态中断
bflb_sdh_sta_int_en(sdh, SDH_NORMAL_STA_CARD_INSERT | SDH_NORMAL_STA_CARD_REMOVE, true);

// 轮询方式检测卡插入
uint32_t sta = bflb_sdh_sta_get(sdh);
if (sta & SDH_NORMAL_STA_CARD_INSERT) {
    printf("Card inserted\r\n");
}

// 获取卡检测引脚状态 (feature control)
int inserted = bflb_sdh_feature_control(sdh, SDH_CMD_GET_PRESENT_STA_CARD_INSERTED, 0);
```

### 4.3 数据传输

```c
int bflb_sdh_tranfer_start(struct bflb_device_s *dev, 
                           struct bflb_sdh_cmd_cfg_s *cmd_cfg, 
                           struct bflb_sdh_data_cfg_s *data_cfg);

int bflb_sdh_get_resp(struct bflb_device_s *dev, struct bflb_sdh_cmd_cfg_s *cmd_cfg);
```

### 4.4 状态管理

```c
uint32_t bflb_sdh_sta_get(struct bflb_device_s *dev);          // 获取状态
void bflb_sdh_sta_clr(struct bflb_device_s *dev, uint32_t sta_bit);  // 清除状态
void bflb_sdh_sta_int_en(struct bflb_device_s *dev, uint32_t sta_bit, bool en);  // 中断使能
```

### 4.5 特性控制

```c
int bflb_sdh_feature_control(struct bflb_device_s *dev, int cmd, uintptr_t arg);

// 常用命令:
SDH_CMD_SET_BUS_WIDTH          // 设置总线宽度
SDH_CMD_SET_HS_MODE_EN        // 高速模式使能
SDH_CMD_SET_BUS_CLK_DIV        // 设置时钟分频
SDH_CMD_SET_INTERNAL_CLK_EN    // 内部时钟使能
SDH_CMD_GET_INTERNAL_CLK_STABLE  // 获取时钟稳定状态
SDH_CMD_SOFT_RESET_ALL        // 软件复位
```

---

## 5. SDIO2 API

SDIO2是SDIO 2.0向后兼容模式。

```c
int bflb_sdio2_init(struct bflb_device_s *dev, uint32_t dnld_size_max);
int bflb_sdio2_deinit(struct bflb_device_s *dev);

// 传输队列
int bflb_sdio2_dnld_port_push(struct bflb_device_s *dev, bflb_sdio2_trans_desc_t *trans_desc);
int bflb_sdio2_upld_port_push(struct bflb_device_s *dev, bflb_sdio2_trans_desc_t *trans_desc);
int bflb_sdio2_dnld_port_pop(struct bflb_device_s *dev, bflb_sdio2_trans_desc_t *trans_desc);
int bflb_sdio2_upld_port_pop(struct bflb_device_s *dev, bflb_sdio2_trans_desc_t *trans_desc);

// 中断回调
int bflb_sdio2_irq_attach(struct bflb_device_s *dev, bflb_sdio2_irq_cb_t irq_event_cb, void *arg);

// 特性控制
int bflb_sdio2_feature_control(struct bflb_device_s *dev, int cmd, uintptr_t arg);
```

---

## 6. SDIO3 API

SDIO3是高性能SDIO 3.0模式，支持SDR104/SDR50/DDR50。

```c
int bflb_sdio3_init(struct bflb_device_s *dev, struct bflb_sdio3_config_s *cfg);
int bflb_sdio3_deinit(struct bflb_device_s *dev);

// 配置结构
struct bflb_sdio3_config_s {
    uint8_t func_num;              // 功能号: 1~2
    uint32_t ocr;                  // 操作电压范围
    uint32_t cap_flag;             // 能力标志
    uint32_t func1_dnld_size_max;  // Function 1下载最大尺寸
    uint32_t func2_dnld_size_max;  // Function 2下载最大尺寸
};

// 传输接口
int bflb_sdio3_dnld_push(struct bflb_device_s *dev, bflb_sdio3_trans_desc_t *trans_desc);
int bflb_sdio3_upld_push(struct bflb_device_s *dev, bflb_sdio3_trans_desc_t *trans_desc);
int bflb_sdio3_dnld_pop(struct bflb_device_s *dev, bflb_sdio3_trans_desc_t *trans_desc, uint8_t func);
int bflb_sdio3_upld_pop(struct bflb_device_s *dev, bflb_sdio3_trans_desc_t *trans_desc, uint8_t func);

// 自定义寄存器
int bflb_sdio3_custom_reg_read(struct bflb_device_s *dev, uint16_t reg_offset, void *buff, uint16_t len);
int bflb_sdio3_custom_reg_write(struct bflb_device_s *dev, uint16_t reg_offset, void *buff, uint16_t len);

// 中断回调
int bflb_sdio3_irq_attach(struct bflb_device_s *dev, bflb_sdio3_irq_cb_t irq_event_cb, void *arg);

// 特性控制
int bflb_sdio3_feature_control(struct bflb_device_s *dev, int cmd, uintptr_t arg);
```

---

## 7. 工作代码示例

### 7.1 SDH初始化与卡检测

```c
#include "bflb_sdh.h"
#include "bflb_irq.h"

static struct bflb_device_s *sdh_dev;

void sdh_isr(int irq, void *arg)
{
    uint32_t sta = bflb_sdh_sta_get(sdh_dev);
    
    if (sta & SDH_NORMAL_STA_CARD_INSERT) {
        printf("Card inserted\r\n");
        bflb_sdh_sta_clr(sdh_dev, SDH_NORMAL_STA_CARD_INSERT);
    }
    if (sta & SDH_NORMAL_STA_CARD_REMOVE) {
        printf("Card removed\r\n");
        bflb_sdh_sta_clr(sdh_dev, SDH_NORMAL_STA_CARD_REMOVE);
    }
    
    /* 处理其他中断... */
}

int sdh_card_init(void)
{
    struct bflb_sdh_config_s cfg = {
        .dma_fifo_th = SDH_DMA_FIFO_THRESHOLD_256,
        .dma_burst = SDH_DMA_BURST_256,
        .power_vol = 0,
    };
    
    sdh_dev = bflb_device_get_by_name("sdh");
    if (!sdh_dev) {
        printf("Get SDH device failed\r\n");
        return -1;
    }
    
    bflb_sdh_init(sdh_dev, &cfg);
    
    /* 注册中断 */
    bflb_irq_register(sdh_dev->irq_num, sdh_isr, NULL);
    bflb_irq_enable(sdh_dev->irq_num);
    
    /* 启用卡插入/移除中断 */
    bflb_sdh_sta_int_en(sdh_dev, 
                        SDH_NORMAL_STA_CARD_INSERT | SDH_NORMAL_STA_CARD_REMOVE, 
                        true);
    
    return 0;
}
```

### 7.2 SDIO3完整初始化

```c
#include "bflb_sdio3.h"

static struct bflb_device_s *sdio3_dev;

void sdio3_irq_callback(void *arg, uint32_t irq_event, bflb_sdio3_trans_desc_t *trans_desc)
{
    switch (irq_event) {
        case SDIO3_IRQ_EVENT_DNLD_CPL:
            printf("Download complete\r\n");
            break;
        case SDIO3_IRQ_EVENT_UPLD_CPL:
            printf("Upload complete\r\n");
            break;
        case SDIO3_IRQ_EVENT_ERR_CRC:
            printf("CRC error\r\n");
            break;
        case SDIO3_IRQ_EVENT_ERR_ADMA:
            printf("ADMA error\r\n");
            break;
        default:
            printf("IRQ event: %u\r\n", irq_event);
            break;
    }
}

int sdio3_init(void)
{
    struct bflb_sdio3_config_s cfg = {
        .func_num = 1,
        .ocr = 0x00300000,  // 3.0V-3.4V
        .cap_flag = SDIO3_CAP_FLAG_SDR104 | SDIO3_CAP_FLAG_SDR50,
        .func1_dnld_size_max = 4096,
        .func2_dnld_size_max = 0,
    };
    
    sdio3_dev = bflb_device_get_by_name("sdio3");
    if (!sdio3_dev) {
        printf("Get SDIO3 device failed\r\n");
        return -1;
    }
    
    int ret = bflb_sdio3_init(sdio3_dev, &cfg);
    if (ret != 0) {
        printf("SDIO3 init failed: %d\r\n", ret);
        return ret;
    }
    
    /* 注册中断回调 */
    bflb_sdio3_irq_attach(sdio3_dev, sdio3_irq_callback, NULL);
    
    return 0;
}
```

### 7.3 数据读取

```c
int sdh_read_blocks(uint32_t start_block, uint16_t block_count, uint8_t *buffer)
{
    struct bflb_sdh_cmd_cfg_s cmd_cfg = {
        .index = 17,  // READ_SINGLE_BLOCK
        .cmd_type = SDH_CMD_TPYE_NORMAL,
        .resp_type = SDH_RESP_TPYE_R1,
        .argument = start_block,
    };
    
    struct bflb_sdh_data_cfg_s data_cfg = {
        .data_dir = SDH_TRANSFER_DIR_RD,
        .data_type = SDH_DATA_TPYE_NORMAL,
        .auto_cmd_mode = SDH_AUTO_CMD_DISABLE,
        .block_size = 512,
        .block_count = block_count,
        .adma2_hw_desc_raw_mode = false,
    };
    
    struct bflb_sdh_data_tranfer_s adma_tranfer = {
        .address = buffer,
        .length = block_count * 512,
        .int_en = true,
    };
    
    data_cfg.adma_tranfer = &adma_tranfer;
    data_cfg.adma_tranfer_cnt = 1;
    
    int ret = bflb_sdh_tranfer_start(sdh_dev, &cmd_cfg, &data_cfg);
    if (ret != 0) {
        printf("Read transfer failed: %d\r\n", ret);
        return ret;
    }
    
    /* 获取响应 */
    bflb_sdh_get_resp(sdh_dev, &cmd_cfg);
    
    return 0;
}
```

### 7.4 数据写入

```c
int sdh_write_blocks(uint32_t start_block, uint16_t block_count, uint8_t *buffer)
{
    struct bflb_sdh_cmd_cfg_s cmd_cfg = {
        .index = 24,  // WRITE_BLOCK
        .cmd_type = SDH_CMD_TPYE_NORMAL,
        .resp_type = SDH_RESP_TPYE_R1,
        .argument = start_block,
    };
    
    struct bflb_sdh_data_cfg_s data_cfg = {
        .data_dir = SDH_TRANSFER_DIR_WR,
        .data_type = SDH_DATA_TPYE_NORMAL,
        .auto_cmd_mode = SDH_AUTO_CMD_DISABLE,
        .block_size = 512,
        .block_count = block_count,
        .adma2_hw_desc_raw_mode = false,
    };
    
    struct bflb_sdh_data_tranfer_s adma_tranfer = {
        .address = buffer,
        .length = block_count * 512,
        .int_en = true,
    };
    
    data_cfg.adma_tranfer = &adma_tranfer;
    data_cfg.adma_tranfer_cnt = 1;
    
    int ret = bflb_sdh_tranfer_start(sdh_dev, &cmd_cfg, &data_cfg);
    if (ret != 0) {
        printf("Write transfer failed: %d\r\n", ret);
        return ret;
    }
    
    bflb_sdh_get_resp(sdh_dev, &cmd_cfg);
    
    return 0;
}
```

### 7.5 SDIO3上传/下载

```c
int sdio3_upload_data(uint8_t func, uint16_t data_len, uint8_t *buffer)
{
    bflb_sdio3_trans_desc_t trans_desc = {
        .func = func,
        .buff_len = data_len,
        .data_len = data_len,
        .buff = buffer,
        .user_arg = NULL,
    };
    
    return bflb_sdio3_upld_push(sdio3_dev, &trans_desc);
}

int sdio3_download_data(uint8_t func, uint16_t data_len, uint8_t *buffer)
{
    bflb_sdio3_trans_desc_t trans_desc = {
        .func = func,
        .buff_len = data_len,
        .data_len = data_len,
        .buff = buffer,
        .user_arg = NULL,
    };
    
    return bflb_sdio3_dnld_push(sdio3_dev, &trans_desc);
}
```

---

## 8. ADMA2描述符表

```
|----------------|---------------|-------------|--------------------------|
| Address field  |     Length    | Reserved    |         Attribute        |
|----------------|---------------|-------------|--------------------------|
|63            32|31           16|15         06|05  |04  |03|02 |01 |00   |
|----------------|---------------|-------------|----|----|--|---|---|-----|
| 32-bit address | 16-bit length | 0000000000  |Act2|Act1| 0|Int|End|Valid|
|----------------|---------------|-------------|----|----|--|---|---|-----|

属性:
- Valid (bit 0): 描述符有效
- End (bit 1): 描述符结束
- Int (bit 2): DMA中断
- Act (bit 4-5): 00=NOP, 01=Reserved, 10=传输数据, 11=链接描述符

最大描述符长度: 64KB (0xFFFF + 1)
```

---

## 9. 寄存器速查

**SDH_BASE = 0x2000C800**

| 偏移 | 名称 | 说明 |
|------|------|------|
| 0x00 | SDH_SYS_ADDR | 系统地址寄存器 |
| 0x04 | SDH_BLOCK_SIZE | 块大小寄存器 |
| 0x06 | SDH_BLOCK_COUNT | 块数量寄存器 |
| 0x08 | SDH_ARGUMENT | 命令参数寄存器 |
| 0x0C | SDH_TRANS_MOD | 传输模式寄存器 |
| 0x0E | SDH_CMD | 命令寄存器 |
| 0x10-0x14 | SDH_RESP | 响应寄存器0-3 |
| 0x18 | SDH_DATA_PORT | 数据端口 |
| 0x1C | SDH_PRESENT_STA | 当前状态 |
| 0x20 | SDH_HOST_CTL | 主机控制 |
| 0x24 | SDH_PWR_CTL | 电源控制 |
| 0x28 | SDH_CLK_CTL | 时钟控制 |
| 0x2C | SDH_TOUT_CTL | 超时控制 |
| 0x30 | SDH_SWRST | 软件复位 |
| 0x34 | SDH_NOR_INTS_STA | 正常中断状态 |
| 0x36 | SDH_ERR_INTS_STA | 错误中断状态 |
| 0x38 | SDH_NOR_INTS_EN | 正常中断使能 |
| 0x3A | SDH_ERR_INTS_EN | 错误中断使能 |
| 0x3C | SDH_NOR_INTS_SIGNAL_EN | 正常中断信号使能 |
| 0x3E | SDH_ERR_INTS_SIGNAL_EN | 错误中断信号使能 |
| 0x40 | SDH_ADMA_ES | ADMA状态 |
| 0x48 | SDH_ADMA_ADDR | ADMA地址 |
