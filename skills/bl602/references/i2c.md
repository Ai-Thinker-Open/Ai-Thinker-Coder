# I2C API 参考

> 来源文件：`components/platform/hosal/include/hosal_i2c.h`

## 宏定义

```c
#define HOSAL_WAIT_FOREVER 0xFFFFFFFFU  // 无限等待

#define HOSAL_I2C_MODE_MASTER 1         // 主机模式
#define HOSAL_I2C_MODE_SLAVE  2        // 从机模式

#define HOSAL_I2C_ADDRESS_WIDTH_7BIT  0  // 7 位地址（常用）
#define HOSAL_I2C_ADDRESS_WIDTH_10BIT 1   // 10 位地址
```

## 类型定义

### `hosal_i2c_config_t` — I2C 配置结构

```c
typedef struct {
    uint32_t address_width;  // 地址宽度：7bit / 10bit
    uint32_t freq;           // I2C 频率（Hz），如 400000 = 400kHz
    uint8_t  scl;            // SCL 引脚
    uint8_t  sda;            // SDA 引脚
    uint8_t  mode;           // 主机/从机模式
} hosal_i2c_config_t;
```

### `hosal_i2c_dev_t` — I2C 设备结构

```c
typedef struct {
    uint8_t       port;       // I2C 端口号 (0/1)
    hosal_i2c_config_t  config;
    void         *priv;
} hosal_i2c_dev_t;
```

## 函数接口

### `hosal_i2c_init`

初始化 I2C。

```c
int hosal_i2c_init(hosal_i2c_dev_t *i2c);
```

---

### `hosal_i2c_master_send`

主机发送数据。

```c
int hosal_i2c_master_send(hosal_i2c_dev_t *i2c,
                          uint16_t dev_addr,
                          const uint8_t *data,
                          uint16_t size,
                          uint32_t timeout);
```

| 参数 | 说明 |
|------|------|
| `i2c` | I2C 设备 |
| `dev_addr` | 从机设备地址（7 位地址） |
| `data` | 发送数据缓冲区 |
| `size` | 发送字节数 |
| `timeout` | 超时（毫秒），`HOSAL_WAIT_FOREVER` 无限等待 |

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_i2c_master_recv`

主机接收数据。

```c
int hosal_i2c_master_recv(hosal_i2c_dev_t *i2c,
                          uint16_t dev_addr,
                          uint8_t *data,
                          uint16_t size,
                          uint32_t timeout);
```

| 参数 | 说明 |
|------|------|
| `i2c` | I2C 设备 |
| `dev_addr` | 从机设备地址（7 位地址） |
| `data` | 接收数据缓冲区 |
| `size` | 期望接收字节数 |
| `timeout` | 超时（毫秒） |

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_i2c_slave_send`

从机发送数据。

```c
int hosal_i2c_slave_send(hosal_i2c_dev_t *i2c,
                         const uint8_t *data,
                         uint16_t size,
                         uint32_t timeout);
```

---

### `hosal_i2c_slave_recv`

从机接收数据。

```c
int hosal_i2c_slave_recv(hosal_i2c_dev_t *i2c,
                         uint8_t *data,
                         uint16_t size,
                         uint32_t timeout);
```

---

### `hosal_i2c_mem_write`

内存写（带寄存器地址，用于访问传感器内部寄存器）。

```c
int hosal_i2c_mem_write(hosal_i2c_dev_t *i2c,
                        uint16_t dev_addr,
                        uint32_t mem_addr,
                        uint16_t mem_addr_size,
                        const uint8_t *data,
                        uint16_t size,
                        uint32_t timeout);
```

| 参数 | 说明 |
|------|------|
| `mem_addr` | 从机内部寄存器地址 |
| `mem_addr_size` | 寄存器地址宽度（1/2/3/4 字节） |

---

### `hosal_i2c_mem_read`

内存读（带寄存器地址）。

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

释放 I2C。

```c
int hosal_i2c_finalize(hosal_i2c_dev_t *i2c);
```

## 使用示例

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

// 主机发送
uint8_t tx_data[2] = {0x30, 0x93};
hosal_i2c_master_send(&i2c0, 0x44, tx_data, 2, HOSAL_WAIT_FOREVER);

// 主机接收
uint8_t rx_buf[6];
hosal_i2c_master_recv(&i2c0, 0x44, rx_buf, 6, HOSAL_WAIT_FOREVER);

// 内存读写（访问传感器寄存器）
uint8_t reg_addr = 0x24;
uint8_t val;
hosal_i2c_mem_write(&i2c0, 0x44, reg_addr, 1, &val, 1, HOSAL_WAIT_FOREVER);
hosal_i2c_mem_read(&i2c0, 0x44, reg_addr, 1, &val, 1, HOSAL_WAIT_FOREVER);
```
