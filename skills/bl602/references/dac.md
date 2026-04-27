# DAC API Reference

> Source file: `components/platform/hosal/include/hosal_dac.h`

## Type Definitions

### `hosal_dac_cb_t` — DAC Callback Function Type

```c
typedef void (*hosal_dac_cb_t)(void *arg);
```

### `hosal_dac_config_t` — DAC Configuration Structure

```c
typedef struct {
    uint8_t  dma_enable;  // 1: use DMA, 0: do not use DMA
    uint32_t pin;         // DAC pin
    uint32_t freq;        // DAC frequency
} hosal_dac_config_t;
```

### `hosal_dac_dev_t` — DAC Device Structure

```c
typedef struct {
    uint8_t            port;       // DAC port number
    hosal_dac_config_t config;    // DAC configuration
    hosal_dac_cb_t     cb;        // DMA callback
    hosal_dma_chan_t   dma_chan;  // DMA channel
    void              *arg;        // Callback argument
    void              *priv;
} hosal_dac_dev_t;
```

## Function API

### `hosal_dac_init`

Initialize DAC.

```c
int hosal_dac_init(hosal_dac_dev_t *dac);
```

| Parameter | Description |
|------|------|
| `dac` | DAC device structure pointer |

**Return value**: `0` success, `EIO` failure

---

### `hosal_dac_finalize`

Release DAC.

```c
int hosal_dac_finalize(hosal_dac_dev_t *dac);
```

| Parameter | Description |
|------|------|
| `dac` | DAC device |

**Return value**: `0` success, `EIO` failure

---

### `hosal_dac_start`

Start DAC output (non-DMA mode).

```c
int hosal_dac_start(hosal_dac_dev_t *dac);
```

---

### `hosal_dac_stop`

Stop DAC output.

```c
int hosal_dac_stop(hosal_dac_dev_t *dac);
```

---

### `hosal_dac_set_value`

Set DAC output value (in μV).

```c
int hosal_dac_set_value(hosal_dac_dev_t *dac, uint32_t data);
```

| Parameter | Description |
|------|------|
| `dac` | DAC device |
| `data` | Output value in μV |

**Return value**: `0` success, `EIO` failure

---

### `hosal_dac_get_value`

Get the most recent DAC output value.

```c
int hosal_dac_get_value(hosal_dac_dev_t *dac);
```

**Return value**: DAC output value (μV)

---

### `hosal_dac_dma_cb_reg`

Register DMA mode completion callback.

```c
int hosal_dac_dma_cb_reg(hosal_dac_dev_t *dac, hosal_dac_cb_t callback, void *arg);
```

---

### `hosal_dac_dma_start`

Start DMA mode DAC output.

```c
int hosal_dac_dma_start(hosal_dac_dev_t *dac, uint32_t *data, uint32_t size);
```

| Parameter | Description |
|------|------|
| `dac` | DAC device |
| `data` | DAC data buffer |
| `size` | Buffer size |

---

### `hosal_dac_dma_stop`

Stop DMA mode DAC output.

```c
int hosal_dac_dma_stop(hosal_dac_dev_t *dac);
```

## Usage Example

```c
#include "hal_dac.h"

hosal_dac_dev_t dac0 = {
    .port = 0,
    .config = {
        .dma_enable = 0,   // Non-DMA mode
        .pin = 0,
        .freq = 0,
    }
};

hosal_dac_init(&dac0);
hosal_dac_start(&dac0);

// Set output voltage value (in μV), e.g., 3300000 = 3.3V
hosal_dac_set_value(&dac0, 3300000);

// Get current output value
int val = hosal_dac_get_value(&dac0);

// DMA mode
hosal_dac_dma_cb_reg(&dac0, my_dac_callback, NULL);
hosal_dac_dma_start(&dac0, dac_buf, sizeof(dac_buf));
```
