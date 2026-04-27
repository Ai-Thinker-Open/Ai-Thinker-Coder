# ADC API 参考

> 来源文件：`components/platform/hosal/include/hosal_adc.h`

## 宏定义

```c
#define HOSAL_WAIT_FOREVER 0xFFFFFFFFU  // 无限等待
```

## 类型定义

### `hosal_adc_sample_mode_t` — 采样模式

```c
typedef enum {
    HOSAL_ADC_ONE_SHOT,   // 单次采样
    HOSAL_ADC_CONTINUE    // 连续采样
} hosal_adc_sample_mode_t;
```

### `hosal_adc_event_t` — ADC 中断事件

```c
typedef enum {
    HOSAL_ADC_INT_OV,      // 溢出错误
    HOSAL_ADC_INT_EOS,     // 采样结束（End of Sample）
    HOSAL_ADC_INT_DMA_TRH, // DMA 传输半满
    HOSAL_ADC_INT_DMA_TRC, // DMA 传输完成
    HOSAL_ADC_INT_DMA_TRE, // DMA 传输错误
} hosal_adc_event_t;
```

### `hosal_adc_config_t` — ADC 配置结构

```c
typedef struct {
    uint32_t sampling_freq;       // 采样频率 Hz
    uint32_t pin;                // ADC 引脚
    hosal_adc_sample_mode_t mode; // 采样模式
    uint8_t  sample_resolution;   // 采样分辨率（位数）
} hosal_adc_config_t;
```

### `hosal_adc_irq_t` — ADC 中断回调类型

```c
typedef void (*hosal_adc_irq_t)(void *parg);
```

### `hosal_adc_cb_t` — ADC 采样回调类型

```c
typedef void (*hosal_adc_cb_t)(hosal_adc_event_t event, void *data, uint32_t size);
```

### `hosal_adc_dev_t` — ADC 设备结构

```c
typedef struct {
    uint8_t port;
    hosal_adc_config_t config;
    hosal_dma_chan_t dma_chan;   // DMA 通道
    hosal_adc_irq_t cb;           // 中断回调
    void *p_arg;
    void *priv;
} hosal_adc_dev_t;
```

## 函数接口

### `hosal_adc_init`

初始化 ADC。

```c
int hosal_adc_init(hosal_adc_dev_t *adc);
```

---

### `hosal_adc_add_channel`

添加一个 ADC 采样通道。

```c
int hosal_adc_add_channel(hosal_adc_dev_t *adc, uint32_t channel);
```

---

### `hosal_adc_remove_channel`

移除一个 ADC 采样通道。

```c
int hosal_adc_remove_channel(hosal_adc_dev_t *adc, uint32_t channel);
```

---

### `hosal_adc_add_reference_channel`

添加参考通道。

```c
int hosal_adc_add_reference_channel(hosal_adc_dev_t *adc,
                                     uint32_t refer_channel,
                                     float refer_voltage);
```

---

### `hosal_adc_remove_reference_channel`

移除参考通道。

```c
int hosal_adc_remove_reference_channel(hosal_adc_dev_t *adc);
```

---

### `hosal_adc_value_get`

读取单次 ADC 采样值（阻塞）。

```c
int hosal_adc_value_get(hosal_adc_dev_t *adc,
                        uint32_t channel,
                        uint32_t timeout);
```

| 参数 | 说明 |
|------|------|
| `adc` | ADC 设备 |
| `channel` | ADC 通道号 |
| `timeout` | 超时（毫秒） |

**返回值**：采样值（>=0），失败 `-1`

---

### `hosal_adc_tsen_value_get`

读取内部温度传感器。

```c
int hosal_adc_tsen_value_get(hosal_adc_dev_t *adc);
```

---

### `hosal_adc_sample_cb_reg`

注册 ADC 采样回调（连续采样模式下使用）。

```c
int hosal_adc_sample_cb_reg(hosal_adc_dev_t *adc, hosal_adc_cb_t cb);
```

---

### `hosal_adc_start`

启动连续采样。

```c
int hosal_adc_start(hosal_adc_dev_t *adc, void *data, uint32_t size);
```

| 参数 | 说明 |
|------|------|
| `data` | 采样数据缓冲区 |
| `size` | 缓冲区大小（按分辨率对齐） |

---

### `hosal_adc_stop`

停止 ADC 采样。

```c
int hosal_adc_stop(hosal_adc_dev_t *adc);
```

---

### `hosal_adc_finalize`

释放 ADC。

```c
int hosal_adc_finalize(hosal_adc_dev_t *adc);
```

## 使用示例

```c
#include "hal_adc.h"

hosal_adc_dev_t adc0 = {
    .port = 0,
    .config = {
        .sampling_freq = 100000,   // 100kHz
        .pin = 0,                 // ADC channel 0
        .mode = HOSAL_ADC_ONE_SHOT,  // 单次模式
        .sample_resolution = 12,  // 12 位
    }
};

hosal_adc_init(&adc0);
hosal_adc_add_channel(&adc0, 0);

// 单次读取
int val = hosal_adc_value_get(&adc0, 0, HOSAL_WAIT_FOREVER);
if (val >= 0) {
    float voltage = val * 3.3f / 4096.0f;
    printf("ADC: %d, Voltage: %.3fV\r\n", val, voltage);
}

// 读取内部温度传感器
int temp = hosal_adc_tsen_value_get(&adc0);

hosal_adc_finalize(&adc0);
```
