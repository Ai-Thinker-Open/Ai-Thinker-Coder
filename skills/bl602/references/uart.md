# UART API 参考

> 来源文件：`components/platform/hosal/include/hosal_uart.h`

## 宏定义

```c
#define HOSAL_UART_AUTOBAUD_0X55     1   // 使用 0x55 自动波特率检测
#define HOSAL_UART_AUTOBAUD_STARTBIT 2   // 使用起始位自动检测

// 回调类型
#define HOSAL_UART_TX_CALLBACK       1   // TX 空闲中断回调
#define HOSAL_UART_RX_CALLBACK       2   // RX 完成回调
#define HOSAL_UART_TX_DMA_CALLBACK   3   // TX DMA 完成回调
#define HOSAL_UART_RX_DMA_CALLBACK   4   // RX DMA 完成回调

// ioctl 控制命令
#define HOSAL_UART_BAUD_SET          1
#define HOSAL_UART_BAUD_GET          2
#define HOSAL_UART_DATA_WIDTH_SET    3
#define HOSAL_UART_DATA_WIDTH_GET    4
#define HOSAL_UART_STOP_BITS_SET     5
#define HOSAL_UART_STOP_BITS_GET     6
#define HOSAL_UART_FLOWMODE_SET      7
#define HOSAL_UART_FLOWSTAT_GET      8
#define HOSAL_UART_PARITY_SET        9
#define HOSAL_UART_PARITY_GET       10
#define HOSAL_UART_MODE_SET         11
#define HOSAL_UART_MODE_GET         12
#define HOSAL_UART_FREE_TXFIFO_GET  13
#define HOSAL_UART_FREE_RXFIFO_GET  14
#define HOSAL_UART_FLUSH            15
#define HOSAL_UART_TX_TRIGGER_ON    16
#define HOSAL_UART_TX_TRIGGER_OFF   17
#define HOSAL_UART_DMA_TX_START     18
#define HOSAL_UART_DMA_RX_START     19
```

## 类型定义

### `hosal_uart_data_width_t` — 数据位宽

```c
typedef enum {
    HOSAL_DATA_WIDTH_5BIT,
    HOSAL_DATA_WIDTH_6BIT,
    HOSAL_DATA_WIDTH_7BIT,
    HOSAL_DATA_WIDTH_8BIT,  // 常用
    HOSAL_DATA_WIDTH_9BIT
} hosal_uart_data_width_t;
```

### `hosal_uart_stop_bits_t` — 停止位

```c
typedef enum {
    HOSAL_STOP_BITS_1,      // 1 位停止位（常用）
    HOSAL_STOP_BITS_1_5,   // 1.5 位停止位
    HOSAL_STOP_BITS_2       // 2 位停止位
} hosal_uart_stop_bits_t;
```

### `hosal_uart_parity_t` — 校验位

```c
typedef enum {
    HOSAL_NO_PARITY,        // 无校验（常用）
    HOSAL_ODD_PARITY,      // 奇校验
    HOSAL_EVEN_PARITY      // 偶校验
} hosal_uart_parity_t;
```

### `hosal_uart_flow_control_t` — 流控制

```c
typedef enum {
    HOSAL_FLOW_CONTROL_DISABLED, // 无流控（常用）
    HOSAL_FLOW_CONTROL_CTS,
    HOSAL_FLOW_CONTROL_RTS,
    HOSAL_FLOW_CONTROL_CTS_RTS
} hosal_uart_flow_control_t;
```

### `hosal_uart_mode_t` — 模式

```c
typedef enum {
    HOSAL_UART_MODE_POLL,      // 轮询模式（默认）
    HOSAL_UART_MODE_INT_TX,    // TX 中断模式
    HOSAL_UART_MODE_INT_RX,    // RX 中断模式
    HOSAL_UART_MODE_INT,       // TX+RX 中断模式
} hosal_uart_mode_t;
```

### `hosal_uart_callback_t` — 回调函数类型

```c
typedef int (*hosal_uart_callback_t)(void *p_arg);
```

### `hosal_uart_dma_cfg_t` — DMA 配置

```c
typedef struct {
    uint8_t *dma_buf;         // DMA 缓冲区
    uint32_t dma_buf_size;     // 缓冲区大小
} hosal_uart_dma_cfg_t;
```

### `hosal_uart_config_t` — UART 配置结构

```c
typedef struct {
    uint8_t                   uart_id;        // UART ID (0/1/2)
    uint8_t                   tx_pin;         // TX 引脚
    uint8_t                   rx_pin;         // RX 引脚
    uint8_t                   cts_pin;        // CTS 引脚（255=不用）
    uint8_t                   rts_pin;        // RTS 引脚（255=不用）
    uint32_t                  baud_rate;      // 波特率
    hosal_uart_data_width_t   data_width;    // 数据位宽
    hosal_uart_parity_t       parity;         // 校验位
    hosal_uart_stop_bits_t    stop_bits;      // 停止位
    hosal_uart_flow_control_t flow_control;   // 流控制
    hosal_uart_mode_t         mode;           // 模式
} hosal_uart_config_t;
```

### `hosal_uart_dev_t` — UART 设备结构

```c
typedef struct {
    uint8_t       port;
    hosal_uart_config_t config;
    hosal_uart_callback_t tx_cb;
    void *p_txarg;
    hosal_uart_callback_t rx_cb;
    void *p_rxarg;
    hosal_uart_callback_t txdma_cb;
    void *p_txdma_arg;
    hosal_uart_callback_t rxdma_cb;
    void *p_rxdma_arg;
    hosal_dma_chan_t dma_tx_chan;
    hosal_dma_chan_t dma_rx_chan;
    void         *priv;
} hosal_uart_dev_t;
```

## 宏

### 快速声明 UART 配置和设备

```c
// 声明配置
HOSAL_UART_CFG_DECL(cfg, id, tx_pin, rx_pin, baud);
// 例: HOSAL_UART_CFG_DECL(my_cfg, 0, 16, 7, 115200);

// 声明设备
HOSAL_UART_DEV_DECL(my_uart, 0, 16, 7, 115200);
```

## 函数接口

### `hosal_uart_init`

初始化 UART。

```c
int hosal_uart_init(hosal_uart_dev_t *uart);
```

### `hosal_uart_init_only_tx`

仅初始化 TX（单向通信）。

```c
int hosal_uart_init_only_tx(hosal_uart_dev_t *uart);
```

### `hosal_uart_send`

轮询方式发送数据。

```c
int hosal_uart_send(hosal_uart_dev_t *uart, const void *txbuf, uint32_t size);
```

| 参数 | 说明 |
|------|------|
| `uart` | UART 设备 |
| `txbuf` | 发送数据缓冲区 |
| `size` | 发送字节数 |

**返回值**：成功发送的字节数（>0），失败 `EIO`

---

### `hosal_uart_receive`

轮询方式接收数据。

```c
int hosal_uart_receive(hosal_uart_dev_t *uart, void *data, uint32_t expect_size);
```

| 参数 | 说明 |
|------|------|
| `uart` | UART 设备 |
| `data` | 接收数据缓冲区 |
| `expect_size` | 期望接收的字节数 |

**返回值**：实际接收的字节数（>0），失败 `EIO`

---

### `hosal_uart_ioctl`

UART IO 控制。

```c
int hosal_uart_ioctl(hosal_uart_dev_t *uart, int ctl, void *p_arg);
```

| ctl 命令 | p_arg 类型 | 说明 |
|----------|-----------|------|
| `HOSAL_UART_BAUD_SET` | `uint32_t *` | 设置波特率 |
| `HOSAL_UART_DATA_WIDTH_SET` | `hosal_uart_data_width_t *` | 设置数据位宽 |
| `HOSAL_UART_STOP_BITS_SET` | `hosal_uart_stop_bits_t *` | 设置停止位 |
| `HOSAL_UART_PARITY_SET` | `hosal_uart_parity_t *` | 设置校验位 |
| `HOSAL_UART_MODE_SET` | `hosal_uart_mode_t *` | 设置模式 |
| `HOSAL_UART_FLUSH` | `NULL` | 等待发送完成 |
| `HOSAL_UART_DMA_TX_START` | `hosal_uart_dma_cfg_t *` | 启动 DMA 发送 |
| `HOSAL_UART_DMA_RX_START` | `hosal_uart_dma_cfg_t *` | 启动 DMA 接收 |

---

### `hosal_uart_callback_set`

设置中断回调。

```c
int hosal_uart_callback_set(hosal_uart_dev_t *uart,
                            int callback_type,
                            hosal_uart_callback_t pfn_callback,
                            void *arg);
```

| callback_type | 说明 |
|---------------|------|
| `HOSAL_UART_TX_CALLBACK` | TX 空闲回调 |
| `HOSAL_UART_RX_CALLBACK` | RX 完成回调 |
| `HOSAL_UART_TX_DMA_CALLBACK` | TX DMA 完成回调 |
| `HOSAL_UART_RX_DMA_CALLBACK` | RX DMA 完成回调 |

---

### `hosal_uart_finalize`

释放 UART。

```c
int hosal_uart_finalize(hosal_uart_dev_t *uart);
```

## 使用示例

```c
#include "hal_uart.h"

hosal_uart_dev_t uart0 = {
    .port = 0,
    .config = {
        .uart_id = 0,
        .tx_pin = 16,
        .rx_pin = 7,
        .baud_rate = 115200,
        .data_width = HOSAL_DATA_WIDTH_8BIT,
        .parity = HOSAL_NO_PARITY,
        .stop_bits = HOSAL_STOP_BITS_1,
        .mode = HOSAL_UART_MODE_POLL,
    }
};

hosal_uart_init(&uart0);

// 发送
uint8_t tx_data[] = "Hello\r\n";
hosal_uart_send(&uart0, tx_data, sizeof(tx_data) - 1);

// 接收（阻塞等待 10 字节）
uint8_t rx_buf[10];
int len = hosal_uart_receive(&uart0, rx_buf, sizeof(rx_buf));

// 动态修改波特率
uint32_t new_baud = 9600;
hosal_uart_ioctl(&uart0, HOSAL_UART_BAUD_SET, &new_baud);
```
