# SPI API 参考

> 来源文件：`components/platform/hosal/include/hosal_spi.h`

## 宏定义

```c
#define HOSAL_SPI_MODE_MASTER 0  // 主机模式
#define HOSAL_SPI_MODE_SLAVE  1  // 从机模式
#define HOSAL_WAIT_FOREVER  0xFFFFFFFFU  // 超时等待
```

## 类型定义

### `hosal_spi_irq_t` — 中断回调类型

```c
typedef void (*hosal_spi_irq_t)(void *parg);
```

### `hosal_spi_config_t` — SPI 配置结构

```c
typedef struct {
    uint8_t mode;           // 主机/从机模式
    uint8_t dma_enable;     // 是否启用 DMA (0=禁用)
    uint8_t polar_phase;    // 极性和相位 (CPOL=0/1, CPHA=0/1)
    uint32_t freq;          // 通信频率 Hz（如 1000000 = 1MHz）
    uint8_t pin_clk;        // CLK 引脚
    uint8_t pin_mosi;       // MOSI 引脚
    uint8_t pin_miso;       // MISO 引脚
} hosal_spi_config_t;
```

### `hosal_spi_dev_t` — SPI 设备结构

```c
typedef struct {
    uint8_t port;
    hosal_spi_config_t  config;
    hosal_spi_irq_t cb;     // 中断回调
    void *p_arg;
    void *priv;
} hosal_spi_dev_t;
```

## 函数接口

### `hosal_spi_init`

初始化 SPI。

```c
int hosal_spi_init(hosal_spi_dev_t *spi);
```

---

### `hosal_spi_send`

仅发送数据（全双工发送端）。

```c
int hosal_spi_send(hosal_spi_dev_t *spi,
                   const uint8_t *data,
                   uint16_t size,
                   uint32_t timeout);
```

---

### `hosal_spi_recv`

仅接收数据。

```c
int hosal_spi_recv(hosal_spi_dev_t *spi,
                   uint8_t *data,
                   uint16_t size,
                   uint32_t timeout);
```

---

### `hosal_spi_send_recv`

发送并接收（常用全双工操作）。

```c
int hosal_spi_send_recv(hosal_spi_dev_t *spi,
                        uint8_t *tx_data,
                        uint8_t *rx_data,
                        uint16_t size,
                        uint32_t timeout);
```

| 参数 | 说明 |
|------|------|
| `spi` | SPI 设备 |
| `tx_data` | 发送数据缓冲区 |
| `rx_data` | 接收数据缓冲区（可与 tx_data 相同） |
| `size` | 发送/接收字节数 |
| `timeout` | 超时（毫秒） |

---

### `hosal_spi_irq_callback_set`

设置中断回调。

```c
int hosal_spi_irq_callback_set(hosal_spi_dev_t *spi,
                               hosal_spi_irq_t pfn,
                               void *p_arg);
```

---

### `hosal_spi_set_cs`

软件控制 CS 片选引脚（仅主机可用）。

```c
int hosal_spi_set_cs(uint8_t pin, uint8_t value);
```

| 参数 | 说明 |
|------|------|
| `pin` | CS 引脚号 |
| `value` | `0` = 低，`1` = 高 |

---

### `hosal_spi_finalize`

释放 SPI。

```c
int hosal_spi_finalize(hosal_spi_dev_t *spi);
```

## 使用示例

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

// 发送并接收（全双工）
uint8_t tx_buf[4] = {0x9F, 0, 0, 0};
uint8_t rx_buf[4];
hosal_spi_send_recv(&spi0, tx_buf, rx_buf, 4, HOSAL_WAIT_FOREVER);

// 单独发送
hosal_spi_send(&spi0, tx_buf, 4, HOSAL_WAIT_FOREVER);

// 软件控制 CS（片选从机）
hosal_spi_set_cs(17, 0);  // CS 低，选中从机
// ... 操作 ...
hosal_spi_set_cs(17, 1);  // CS 高，释放从机
```
