# SPI API Reference

> Source file: `components/platform/hosal/include/hosal_spi.h`

## Macros

```c
#define HOSAL_SPI_MODE_MASTER 0  // Master mode
#define HOSAL_SPI_MODE_SLAVE  1  // Slave mode
#define HOSAL_WAIT_FOREVER  0xFFFFFFFFU  // Wait forever (no timeout)
```

## Type Definitions

### `hosal_spi_irq_t` — Interrupt Callback Type

```c
typedef void (*hosal_spi_irq_t)(void *parg);
```

### `hosal_spi_config_t` — SPI Configuration Structure

```c
typedef struct {
    uint8_t mode;           // Master/slave mode
    uint8_t dma_enable;     // DMA enable (0=disabled)
    uint8_t polar_phase;    // Polarity and phase (CPOL=0/1, CPHA=0/1)
    uint32_t freq;          // Communication frequency in Hz (e.g. 1000000 = 1MHz)
    uint8_t pin_clk;        // CLK pin
    uint8_t pin_mosi;       // MOSI pin
    uint8_t pin_miso;       // MISO pin
} hosal_spi_config_t;
```

### `hosal_spi_dev_t` — SPI Device Structure

```c
typedef struct {
    uint8_t port;
    hosal_spi_config_t  config;
    hosal_spi_irq_t cb;     // Interrupt callback
    void *p_arg;
    void *priv;
} hosal_spi_dev_t;
```

## Function API

### `hosal_spi_init`

Initialize SPI.

```c
int hosal_spi_init(hosal_spi_dev_t *spi);
```

---

### `hosal_spi_send`

Send data only (full-duplex transmit side).

```c
int hosal_spi_send(hosal_spi_dev_t *spi,
                   const uint8_t *data,
                   uint16_t size,
                   uint32_t timeout);
```

---

### `hosal_spi_recv`

Receive data only.

```c
int hosal_spi_recv(hosal_spi_dev_t *spi,
                   uint8_t *data,
                   uint16_t size,
                   uint32_t timeout);
```

---

### `hosal_spi_send_recv`

Send and receive (common full-duplex operation).

```c
int hosal_spi_send_recv(hosal_spi_dev_t *spi,
                        uint8_t *tx_data,
                        uint8_t *rx_data,
                        uint16_t size,
                        uint32_t timeout);
```

| Parameter | Description |
|-----------|-------------|
| `spi` | SPI device |
| `tx_data` | Transmit data buffer |
| `rx_data` | Receive data buffer (can be same as tx_data) |
| `size` | Number of bytes to send/receive |
| `timeout` | Timeout (milliseconds) |

---

### `hosal_spi_irq_callback_set`

Set interrupt callback.

```c
int hosal_spi_irq_callback_set(hosal_spi_dev_t *spi,
                               hosal_spi_irq_t pfn,
                               void *p_arg);
```

---

### `hosal_spi_set_cs`

Software control of CS chip select pin (master only).

```c
int hosal_spi_set_cs(uint8_t pin, uint8_t value);
```

| Parameter | Description |
|-----------|-------------|
| `pin` | CS pin number |
| `value` | `0` = low, `1` = high |

---

### `hosal_spi_finalize`

Finalize SPI.

```c
int hosal_spi_finalize(hosal_spi_dev_t *spi);
```

## Usage Example

```c
#include "hal_spi.h"

hosal_spi_dev_t spi0 = {
    .port = 0,
    .config = {
        .mode = HOSAL_SPI_MODE_MASTER,
        .dma_enable = 0,
        .polar_phase = 0,      // CPOL=0, CPHA=0
        .freq = 1000000,       // 1MHz
        .pin_clk = 14,
        .pin_mosi = 15,
        .pin_miso = 16,
    }
};

hosal_spi_init(&spi0);

// Send and receive (full-duplex)
uint8_t tx_buf[4] = {0x9F, 0, 0, 0};
uint8_t rx_buf[4];
hosal_spi_send_recv(&spi0, tx_buf, rx_buf, 4, HOSAL_WAIT_FOREVER);

// Send only
hosal_spi_send(&spi0, tx_buf, 4, HOSAL_WAIT_FOREVER);

// Software CS control (chip select slave)
hosal_spi_set_cs(17, 0);  // CS low, select slave
// ... operations ...
hosal_spi_set_cs(17, 1);  // CS high, release slave
```
