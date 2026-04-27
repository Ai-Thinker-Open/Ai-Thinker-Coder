# DAC API 参考

> 来源文件：`components/platform/hosal/include/hosal_dac.h`

## 类型定义

### `hosal_dac_cb_t` — DAC 回调函数类型

```c
typedef void (*hosal_dac_cb_t)(void *arg);
```

### `hosal_dac_config_t` — DAC 配置结构

```c
typedef struct {
    uint8_t  dma_enable;  // 1: 使用 DMA，0: 不使用 DMA
    uint32_t pin;         // DAC 引脚
    uint32_t freq;        // DAC 频率
} hosal_dac_config_t;
```

### `hosal_dac_dev_t` — DAC 设备结构

```c
typedef struct {
    uint8_t            port;       // DAC 端口号
    hosal_dac_config_t config;    // DAC 配置
    hosal_dac_cb_t     cb;        // DMA 回调
    hosal_dma_chan_t   dma_chan;  // DMA 通道
    void              *arg;        // 回调参数
    void              *priv;
} hosal_dac_dev_t;
```

## 函数接口

### `hosal_dac_init`

初始化 DAC。

```c
int hosal_dac_init(hosal_dac_dev_t *dac);
```

| 参数 | 说明 |
|------|------|
| `dac` | DAC 设备结构体指针 |

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_dac_finalize`

释放 DAC。

```c
int hosal_dac_finalize(hosal_dac_dev_t *dac);
```

| 参数 | 说明 |
|------|------|
| `dac` | DAC 设备 |

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_dac_start`

启动 DAC 输出（非 DMA 模式）。

```c
int hosal_dac_start(hosal_dac_dev_t *dac);
```

---

### `hosal_dac_stop`

停止 DAC 输出。

```c
int hosal_dac_stop(hosal_dac_dev_t *dac);
```

---

### `hosal_dac_set_value`

设置 DAC 输出值（μV 为单位）。

```c
int hosal_dac_set_value(hosal_dac_dev_t *dac, uint32_t data);
```

| 参数 | 说明 |
|------|------|
| `dac` | DAC 设备 |
| `data` | 输出值，单位 μV |

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_dac_get_value`

获取最近一次 DAC 输出值。

```c
int hosal_dac_get_value(hosal_dac_dev_t *dac);
```

**返回值**：DAC 输出值（μV）

---

### `hosal_dac_dma_cb_reg`

注册 DMA 模式完成回调。

```c
int hosal_dac_dma_cb_reg(hosal_dac_dev_t *dac, hosal_dac_cb_t callback, void *arg);
```

---

### `hosal_dac_dma_start`

启动 DMA 模式 DAC 输出。

```c
int hosal_dac_dma_start(hosal_dac_dev_t *dac, uint32_t *data, uint32_t size);
```

| 参数 | 说明 |
|------|------|
| `dac` | DAC 设备 |
| `data` | DAC 数据缓冲区 |
| `size` | 缓冲区大小 |

---

### `hosal_dac_dma_stop`

停止 DMA 模式 DAC 输出。

```c
int hosal_dac_dma_stop(hosal_dac_dev_t *dac);
```

## 使用示例

```c
#include "hal_dac.h"

hosal_dac_dev_t dac0 = {
    .port = 0,
    .config = {
        .dma_enable = 0,   // 非 DMA 模式
        .pin = 0,
        .freq = 0,
    }
};

hosal_dac_init(&dac0);
hosal_dac_start(&dac0);

// 设置输出电压值（单位 μV），如 3300000 = 3.3V
hosal_dac_set_value(&dac0, 3300000);

// 获取当前输出值
int val = hosal_dac_get_value(&dac0);

// DMA 模式
hosal_dac_dma_cb_reg(&dac0, my_dac_callback, NULL);
hosal_dac_dma_start(&dac0, dac_buf, sizeof(dac_buf));
```
