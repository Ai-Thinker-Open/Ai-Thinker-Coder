# I2C API Reference

> Source file: `components/platform/hosal/include/hosal_i2c.h`

## Macro Definitions

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
