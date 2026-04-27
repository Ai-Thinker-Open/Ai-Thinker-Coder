# UART API Reference

> Source file: `components/platform/hosal/include/hosal_uart.h`

## Macros

```c
#define HOSAL_UART_AUTOBAUD_0X55     1   // Auto baud detection using 0x55
#define HOSAL_UART_AUTOBAUD_STARTBIT 2   // Auto detection using start bit

// Callback types
#define HOSAL_UART_TX_CALLBACK       1   // TX idle interrupt callback
#define HOSAL_UART_RX_CALLBACK       2   // RX complete callback
#define HOSAL_UART_TX_DMA_CALLBACK   3   // TX DMA complete callback
#define HOSAL_UART_RX_DMA_CALLBACK   4   // RX DMA complete callback

// ioctl control commands
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

## Type Definitions

### `hosal_uart_data_width_t` — Data Width

```c
typedef enum {
    HOSAL_DATA_WIDTH_5BIT,
    HOSAL_DATA_WIDTH_6BIT,
    HOSAL_DATA_WIDTH_7BIT,
    HOSAL_DATA_WIDTH_8BIT,  // Common
    HOSAL_DATA_WIDTH_9BIT
} hosal_uart_data_width_t;
```

### `hosal_uart_stop_bits_t` — Stop Bits

```c
typedef enum {
    HOSAL_STOP_BITS_1,      // 1 stop bit (common)
    HOSAL_STOP_BITS_1_5,   // 1.5 stop bits
    HOSAL_STOP_BITS_2       // 2 stop bits
} hosal_uart_stop_bits_t;
```

### `hosal_uart_parity_t` — Parity

```c
typedef enum {
    HOSAL_NO_PARITY,        // No parity (common)
    HOSAL_ODD_PARITY,      // Odd parity
    HOSAL_EVEN_PARITY      // Even parity
} hosal_uart_parity_t;
```

### `hosal_uart_flow_control_t` — Flow Control

```c
typedef enum {
    HOSAL_FLOW_CONTROL_DISABLED, // No flow control (common)
    HOSAL_FLOW_CONTROL_CTS,
    HOSAL_FLOW_CONTROL_RTS,
    HOSAL_FLOW_CONTROL_CTS_RTS
} hosal_uart_flow_control_t;
```

### `hosal_uart_mode_t` — Mode

```c
typedef enum {
    HOSAL_UART_MODE_POLL,      // Polling mode (default)
    HOSAL_UART_MODE_INT_TX,    // TX interrupt mode
    HOSAL_UART_MODE_INT_RX,    // RX interrupt mode
    HOSAL_UART_MODE_INT,       // TX+RX interrupt mode
} hosal_uart_mode_t;
```

### `hosal_uart_callback_t` — Callback Function Type

```c
typedef int (*hosal_uart_callback_t)(void *p_arg);
```

### `hosal_uart_dma_cfg_t` — DMA Configuration

```c
typedef struct {
    uint8_t *dma_buf;         // DMA buffer
    uint32_t dma_buf_size;     // Buffer size
} hosal_uart_dma_cfg_t;
```

### `hosal_uart_config_t` — UART Configuration Structure

```c
typedef struct {
    uint8_t                   uart_id;        // UART ID (0/1/2)
    uint8_t                   tx_pin;         // TX pin
    uint8_t                   rx_pin;         // RX pin
    uint8_t                   cts_pin;        // CTS pin (255=not used)
    uint8_t                   rts_pin;        // RTS pin (255=not used)
    uint32_t                  baud_rate;      // Baud rate
    hosal_uart_data_width_t   data_width;    // Data width
    hosal_uart_parity_t       parity;         // Parity
    hosal_uart_stop_bits_t    stop_bits;      // Stop bits
    hosal_uart_flow_control_t flow_control;   // Flow control
    hosal_uart_mode_t         mode;           // Mode
} hosal_uart_config_t;
```

### `hosal_uart_dev_t` — UART Device Structure

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

## Macros

### Quick UART Configuration and Device Declaration

```c
// Declare configuration
HOSAL_UART_CFG_DECL(cfg, id, tx_pin, rx_pin, baud);
// Example: HOSAL_UART_CFG_DECL(my_cfg, 0, 16, 7, 115200);

// Declare device
HOSAL_UART_DEV_DECL(my_uart, 0, 16, 7, 115200);
```

## Function API

### `hosal_uart_init`

Initialize UART.

```c
int hosal_uart_init(hosal_uart_dev_t *uart);
```

### `hosal_uart_init_only_tx`

Initialize TX only (one-way communication).

```c
int hosal_uart_init_only_tx(hosal_uart_dev_t *uart);
```

### `hosal_uart_send`

Send data in polling mode.

```c
int hosal_uart_send(hosal_uart_dev_t *uart, const void *txbuf, uint32_t size);
```

| Parameter | Description |
|-----------|-------------|
| `uart` | UART device |
| `txbuf` | Transmit data buffer |
| `size` | Number of bytes to send |

**Return value**: bytes sent (>0) on success, `EIO` on failure

---

### `hosal_uart_receive`

Receive data in polling mode.

```c
int hosal_uart_receive(hosal_uart_dev_t *uart, void *data, uint32_t expect_size);
```

| Parameter | Description |
|-----------|-------------|
| `uart` | UART device |
| `data` | Receive data buffer |
| `expect_size` | Expected number of bytes to receive |

**Return value**: bytes received (>0) on success, `EIO` on failure

---

### `hosal_uart_ioctl`

UART IO control.

```c
int hosal_uart_ioctl(hosal_uart_dev_t *uart, int ctl, void *p_arg);
```

| ctl command | p_arg type | Description |
|-------------|------------|-------------|
| `HOSAL_UART_BAUD_SET` | `uint32_t *` | Set baud rate |
| `HOSAL_UART_DATA_WIDTH_SET` | `hosal_uart_data_width_t *` | Set data width |
| `HOSAL_UART_STOP_BITS_SET` | `hosal_uart_stop_bits_t *` | Set stop bits |
| `HOSAL_UART_PARITY_SET` | `hosal_uart_parity_t *` | Set parity |
| `HOSAL_UART_MODE_SET` | `hosal_uart_mode_t *` | Set mode |
| `HOSAL_UART_FLUSH` | `NULL` | Wait for TX complete |
| `HOSAL_UART_DMA_TX_START` | `hosal_uart_dma_cfg_t *` | Start DMA TX |
| `HOSAL_UART_DMA_RX_START` | `hosal_uart_dma_cfg_t *` | Start DMA RX |

---

### `hosal_uart_callback_set`

Set interrupt callback.

```c
int hosal_uart_callback_set(hosal_uart_dev_t *uart,
                            int callback_type,
                            hosal_uart_callback_t pfn_callback,
                            void *arg);
```

| callback_type | Description |
|---------------|-------------|
| `HOSAL_UART_TX_CALLBACK` | TX idle callback |
| `HOSAL_UART_RX_CALLBACK` | RX complete callback |
| `HOSAL_UART_TX_DMA_CALLBACK` | TX DMA complete callback |
| `HOSAL_UART_RX_DMA_CALLBACK` | RX DMA complete callback |

---

### `hosal_uart_finalize`

Finalize UART.

```c
int hosal_uart_finalize(hosal_uart_dev_t *uart);
```

## Usage Example

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

// Send
uint8_t tx_data[] = "Hello\r\n";
hosal_uart_send(&uart0, tx_data, sizeof(tx_data) - 1);

// Receive (blocking, wait for 10 bytes)
uint8_t rx_buf[10];
int len = hosal_uart_receive(&uart0, rx_buf, sizeof(rx_buf));

// Dynamically change baud rate
uint32_t new_baud = 9600;
hosal_uart_ioctl(&uart0, HOSAL_UART_BAUD_SET, &new_baud);
```
