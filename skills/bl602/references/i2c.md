# I2C API Reference

> Source file: `components/platform/hosal/include/hosal_i2c.h`  
> Register Header: `components/platform/soc/bl602/bl602_std/.../Device/Bouffalo/BL602/Peripherals/i2c_reg.h`  
> Base Address: `I2C_BASE = 0x4000A300`

---

## Register Overview

| Offset | Register Name | Description |
|---|---|---|
| `0x00` | `I2C_CONFIG` | I2C enable, direction, deglitch, SCL sync, sub-address, slave addr, pkt len |
| `0x04` | `I2C_INT_STATUS` | I2C interrupt status flags |
| `0x08` | `I2C_FIFO_CONFIG` | FIFO configuration (RX/TX threshold, DMA enable) |
| `0x0C` | `I2C_FIFO_STATUS` | FIFO status (RX/TX count, full/empty flags) |
| `0x10` | `I2C_FIFO_WDATA` | FIFO write data |
| `0x14` | `I2C_FIFO_RDATA` | FIFO read data |
| `0x18` | `I2C_ADDR_CONFIG` | I2C slave address configuration |
| `0x1C` | `I2C_MATCH_ADDR` | I2C address match configuration |

---

## Key Register Fields

### `I2C_CONFIG` (offset `0x00`)

| Field | Bits | R/W | Description |
|---|---|---|---|
| `I2C_CR_I2C_M_EN` | [0] | RW | I2C master enable |
| `I2C_CR_I2C_PKT_DIR` | [1] | RW | 0=write (master to slave), 1=read (slave to master) |
| `I2C_CR_I2C_DEG_EN` | [2] | RW | Deglitch enable |
| `I2C_CR_I2C_SCL_SYNC_EN` | [3] | RW | SCL synchronization enable |
| `I2C_CR_I2C_SUB_ADDR_EN` | [4] | RW | Sub-address (register address) enable |
| `I2C_CR_I2C_SUB_ADDR_BC` | [6:5] | RW | Sub-address byte count (0–3 bytes) |
| `I2C_CR_I2C_SLV_ADDR` | [14:8] | RW | Slave address (7-bit) |
| `I2C_CR_I2C_PKT_LEN` | [23:16] | RW | Packet data byte count |

### `I2C_INT_STATUS` (offset `0x04`)

| Field | Bits | R/W | Description |
|---|---|---|---|
| `I2C_INT_RX_FULL` | [0] | RO | RX FIFO full |
| `I2C_INT_TX_EMPTY` | [1] | RO | TX FIFO empty |
| `I2C_INT_MATCH` | [2] | RO | Address match |
| `I2C_INT_NACK` | [3] | RO | NACK received |
| `I2C_INT_END` | [4] | RO | Transfer end |
| `I2C_INT_ERR` | [5] | RO | Error (bus arbitration lost, NACK, timeout) |

### `I2C_FIFO_WDATA` (offset `0x10`)

Write data bytes here to send. Write `I2C_CR_I2C_PKT_LEN` bytes for a master transmit operation.

### `I2C_FIFO_RDATA` (offset `0x14`)

Read data bytes here after a master receive operation completes.

---

## Register-Level Programming Sequence

### Master Send (write to slave)

```c
#define I2C_BASE  0x4000A300

static inline void i2c_write_reg(uint32_t base, uint16_t reg_addr,
                                 uint8_t reg_addr_len,
                                 const uint8_t *data, uint8_t len)
{
    volatile uint32_t *cfg   = (volatile uint32_t *)(base + 0x00);
    volatile uint32_t *sts   = (volatile uint32_t *)(base + 0x04);
    volatile uint32_t *fifo  = (volatile uint32_t *)(base + 0x10);

    // Wait for idle
    while (*cfg & 1) { }

    // Configure: master, enable, write direction, sub-addr bytes
    uint32_t config = (1 << 0)   // I2C master enable
                     | (0 << 1)   // write direction
                     | (1 << 4)   // sub-addr enable
                     | ((reg_addr_len - 1) << 5)  // sub-addr byte count
                     | ((len & 0xFF) << 16);      // pkt len
    *cfg = config;

    // Write sub-address bytes first
    for (int i = reg_addr_len - 1; i >= 0; i--) {
        // MSB first
    }

    // Write data to TX FIFO
    for (int i = 0; i < len; i++) {
        *fifo = data[i];
    }

    // Wait for end
    while (!(*sts & (1 << 4))) { }
}
```

### Master Receive (read from slave)

```c
static inline void i2c_read_reg(uint32_t base, uint16_t reg_addr,
                                 uint8_t reg_addr_len,
                                 uint8_t *data, uint8_t len)
{
    volatile uint32_t *cfg   = (volatile uint32_t *)(base + 0x00);
    volatile uint32_t *sts   = (volatile uint32_t *)(base + 0x04);
    volatile uint32_t *fifo  = (volatile uint32_t *)(base + 0x14);

    while (*cfg & 1) { }

    // Configure: master, enable, read direction
    uint32_t config = (1 << 0)   // master enable
                     | (1 << 1)   // read direction
                     | (1 << 4)   // sub-addr enable
                     | ((reg_addr_len - 1) << 5)
                     | ((len & 0xFF) << 16);
    *cfg = config;

    // Wait for end, then read RX FIFO
    while (!(*sts & (1 << 4))) { }
    for (int i = 0; i < len; i++) {
        data[i] = *fifo;
    }
}
```

---

## Usage Examples

```c
#define HOSAL_WAIT_FOREVER 0xFFFFFFFFU  // Wait forever

#define HOSAL_I2C_MODE_MASTER 1         // Master mode
#define HOSAL_I2C_MODE_SLAVE  2        // Slave mode

#define HOSAL_I2C_ADDRESS_WIDTH_7BIT  0  // 7-bit address (common)
#define HOSAL_I2C_ADDRESS_WIDTH_10BIT 1   // 10-bit address
```

## Type Definitions

### `hosal_i2c_config_t` — I2C Configuration Structure

```c
typedef struct {
    uint32_t address_width;  // Address width: 7bit / 10bit
    uint32_t freq;           // I2C frequency (Hz), e.g., 400000 = 400kHz
    uint8_t  scl;            // SCL pin
    uint8_t  sda;            // SDA pin
    uint8_t  mode;           // Master/slave mode
} hosal_i2c_config_t;
```

### `hosal_i2c_dev_t` — I2C Device Structure

```c
typedef struct {
    uint8_t       port;       // I2C port number (0/1)
    hosal_i2c_config_t  config;
    void         *priv;
} hosal_i2c_dev_t;
```

## Function Interface

### `hosal_i2c_init`

Initializes I2C.

```c
int hosal_i2c_init(hosal_i2c_dev_t *i2c);
```

---

### `hosal_i2c_master_send`

Master sends data.

```c
int hosal_i2c_master_send(hosal_i2c_dev_t *i2c,
                          uint16_t dev_addr,
                          const uint8_t *data,
                          uint16_t size,
                          uint32_t timeout);
```

| Parameter | Description |
|-----------|-------------|
| `i2c` | I2C device |
| `dev_addr` | Slave device address (7-bit address) |
| `data` | Send data buffer |
| `size` | Number of bytes to send |
| `timeout` | Timeout (milliseconds), `HOSAL_WAIT_FOREVER` for infinite wait |

**Return value**: `0` success, `EIO` failure

---

### `hosal_i2c_master_recv`

Master receives data.

```c
int hosal_i2c_master_recv(hosal_i2c_dev_t *i2c,
                          uint16_t dev_addr,
                          uint8_t *data,
                          uint16_t size,
                          uint32_t timeout);
```

| Parameter | Description |
|-----------|-------------|
| `i2c` | I2C device |
| `dev_addr` | Slave device address (7-bit address) |
| `data` | Receive data buffer |
| `size` | Expected number of bytes to receive |
| `timeout` | Timeout (milliseconds) |

**Return value**: `0` success, `EIO` failure

---

### `hosal_i2c_slave_send`

Slave sends data.

```c
int hosal_i2c_slave_send(hosal_i2c_dev_t *i2c,
                         const uint8_t *data,
                         uint16_t size,
                         uint32_t timeout);
```

---

### `hosal_i2c_slave_recv`

Slave receives data.

```c
int hosal_i2c_slave_recv(hosal_i2c_dev_t *i2c,
                         uint8_t *data,
                         uint16_t size,
                         uint32_t timeout);
```

---

### `hosal_i2c_mem_write`

Memory write (with register address, used for accessing sensor internal registers).

```c
int hosal_i2c_mem_write(hosal_i2c_dev_t *i2c,
                        uint16_t dev_addr,
                        uint32_t mem_addr,
                        uint16_t mem_addr_size,
                        const uint8_t *data,
                        uint16_t size,
                        uint32_t timeout);
```

| Parameter | Description |
|-----------|-------------|
| `mem_addr` | Slave internal register address |
| `mem_addr_size` | Register address width (1/2/3/4 bytes) |

---

### `hosal_i2c_mem_read`

Memory read (with register address).

```c
int hosal_i2c_mem_read(hosal_i2c_dev_t *i2c,
                       uint16_t dev_addr,
                       uint32_t mem_addr,
                       uint16_t mem_addr_size,
                       uint8_t *data,
                       uint16_t size,
                       uint32_t timeout);
```

---

### `hosal_i2c_finalize`

Releases I2C.

```c
int hosal_i2c_finalize(hosal_i2c_dev_t *i2c);
```

## Usage Example

```c
#include "hal_i2c.h"

hosal_i2c_dev_t i2c0 = {
    .port = 0,
    .config = {
        .address_width = HOSAL_I2C_ADDRESS_WIDTH_7BIT,
        .freq = 400000,    // 400kHz
        .scl = 10,
        .sda = 11,
        .mode = HOSAL_I2C_MODE_MASTER,
    }
};

hosal_i2c_init(&i2c0);

// Master send
uint8_t tx_data[2] = {0x30, 0x93};
hosal_i2c_master_send(&i2c0, 0x44, tx_data, 2, HOSAL_WAIT_FOREVER);

// Master receive
uint8_t rx_buf[6];
hosal_i2c_master_recv(&i2c0, 0x44, rx_buf, 6, HOSAL_WAIT_FOREVER);

// Memory read/write (access sensor registers)
uint8_t reg_addr = 0x24;
uint8_t val;
hosal_i2c_mem_write(&i2c0, 0x44, reg_addr, 1, &val, 1, HOSAL_WAIT_FOREVER);
hosal_i2c_mem_read(&i2c0, 0x44, reg_addr, 1, &val, 1, HOSAL_WAIT_FOREVER);
```
