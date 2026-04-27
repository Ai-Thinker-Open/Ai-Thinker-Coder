# ADC API Reference

> Source file: `components/platform/hosal/include/hosal_adc.h`

## Macro Definitions

```c
#define HOSAL_WAIT_FOREVER 0xFFFFFFFFU  // Wait indefinitely
```

## Type Definitions

### `hosal_adc_sample_mode_t` — Sampling Mode

```c
typedef enum {
    HOSAL_ADC_ONE_SHOT,   // Single sampling
    HOSAL_ADC_CONTINUE    // Continuous sampling
} hosal_adc_sample_mode_t;
```

### `hosal_adc_event_t` — ADC Interrupt Events

```c
typedef enum {
    HOSAL_ADC_INT_OV,      // Overflow error
    HOSAL_ADC_INT_EOS,     // End of Sample
    HOSAL_ADC_INT_DMA_TRH, // DMA transfer half-full
    HOSAL_ADC_INT_DMA_TRC, // DMA transfer complete
    HOSAL_ADC_INT_DMA_TRE, // DMA transfer error
} hosal_adc_event_t;
```

### `hosal_adc_config_t` — ADC Configuration Structure

```c
typedef struct {
    uint32_t sampling_freq;       // Sampling frequency in Hz
    uint32_t pin;                 // ADC pin
    hosal_adc_sample_mode_t mode; // Sampling mode
    uint8_t  sample_resolution;   // Sampling resolution (bits)
} hosal_adc_config_t;
```

### `hosal_adc_irq_t` — ADC Interrupt Callback Type

```c
typedef void (*hosal_adc_irq_t)(void *parg);
```

### `hosal_adc_cb_t` — ADC Sampling Callback Type

```c
typedef void (*hosal_adc_cb_t)(hosal_adc_event_t event, void *data, uint32_t size);
```

### `hosal_adc_dev_t` — ADC Device Structure

```c
typedef struct {
    uint8_t port;
    hosal_adc_config_t config;
    hosal_dma_chan_t dma_chan;   // DMA channel
    hosal_adc_irq_t cb;          // Interrupt callback
    void *p_arg;
    void *priv;
} hosal_adc_dev_t;
```

## Function API

### `hosal_adc_init`

Initialize the ADC.

```c
int hosal_adc_init(hosal_adc_dev_t *adc);
```

---

### `hosal_adc_add_channel`

Add an ADC sampling channel.

```c
int hosal_adc_add_channel(hosal_adc_dev_t *adc, uint32_t channel);
```

---

### `hosal_adc_remove_channel`

Remove an ADC sampling channel.

```c
int hosal_adc_remove_channel(hosal_adc_dev_t *adc, uint32_t channel);
```

---

### `hosal_adc_add_reference_channel`

Add a reference channel.

```c
int hosal_adc_add_reference_channel(hosal_adc_dev_t *adc,
                                     uint32_t refer_channel,
                                     float refer_voltage);
```

---

### `hosal_adc_remove_reference_channel`

Remove the reference channel.

```c
int hosal_adc_remove_reference_channel(hosal_adc_dev_t *adc);
```

---

### `hosal_adc_value_get`

Read a single ADC sample value (blocking).

```c
int hosal_adc_value_get(hosal_adc_dev_t *adc,
                        uint32_t channel,
                        uint32_t timeout);
```

| Parameter | Description |
|-----------|-------------|
| `adc` | ADC device |
| `channel` | ADC channel number |
| `timeout` | Timeout in milliseconds |

**Return value**: Sample value (>=0), failure `-1`

---

### `hosal_adc_tsen_value_get`

Read the internal temperature sensor.

```c
int hosal_adc_tsen_value_get(hosal_adc_dev_t *adc);
```

---

### `hosal_adc_sample_cb_reg`

Register an ADC sampling callback (used in continuous sampling mode).

```c
int hosal_adc_sample_cb_reg(hosal_adc_dev_t *adc, hosal_adc_cb_t cb);
```

---

### `hosal_adc_start`

Start continuous sampling.

```c
int hosal_adc_start(hosal_adc_dev_t *adc, void *data, uint32_t size);
```

| Parameter | Description |
|-----------|-------------|
| `data` | Sampling data buffer |
| `size` | Buffer size (aligned by resolution) |

---

### `hosal_adc_stop`

Stop ADC sampling.

```c
int hosal_adc_stop(hosal_adc_dev_t *adc);
```

---

### `hosal_adc_finalize`

Release the ADC.

```c
int hosal_adc_finalize(hosal_adc_dev_t *adc);
```

## Usage Example

```c
#include "hal_adc.h"

hosal_adc_dev_t adc0 = {
    .port = 0,
    .config = {
        .sampling_freq = 100000,   // 100kHz
        .pin = 0,                  // ADC channel 0
        .mode = HOSAL_ADC_ONE_SHOT, // One-shot mode
        .sample_resolution = 12,    // 12 bits
    }
};

hosal_adc_init(&adc0);
hosal_adc_add_channel(&adc0, 0);

// Single read
int val = hosal_adc_value_get(&adc0, 0, HOSAL_WAIT_FOREVER);
if (val >= 0) {
    float voltage = val * 3.3f / 4096.0f;
    printf("ADC: %d, Voltage: %.3fV\r\n", val, voltage);
}

// Read internal temperature sensor
int temp = hosal_adc_tsen_value_get(&adc0);

hosal_adc_finalize(&adc0);
```
