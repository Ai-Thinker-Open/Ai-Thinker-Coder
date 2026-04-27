# DMA API Reference

> Source file: `components/platform/hosal/include/hosal_dma.h`

## Macro Definitions

```c
#define HOSAL_DMA_INT_TRANS_COMPLETE 0  // Transfer complete interrupt
#define HOSAL_DMA_INT_TRANS_ERROR    1  // Transfer error interrupt
```

## Type Definitions

### `hosal_dma_irq_t` — DMA Interrupt Callback Type

```c
typedef void (*hosal_dma_irq_t)(void *p_arg, uint32_t flag);
```

- `flag` = `0` (`HOSAL_DMA_INT_TRANS_COMPLETE`): Transfer complete
- `flag` = `1` (`HOSAL_DMA_INT_TRANS_ERROR`): Transfer error

### `hosal_dma_chan_t` — DMA Channel Number

```c
typedef int hosal_dma_chan_t;
```

Channel numbers are integers (e.g., 0, 1, 2, etc.).

### `hosal_dma_dev_t` — DMA Device Structure

```c
typedef struct hosal_dma_dev {
    int max_chans;                    // Maximum number of channels
    struct hosal_dma_chan *used_chan; // Array of used channels
    void *priv;
} hosal_dma_dev_t;
```

## Function API

### `hosal_dma_init`

Initialize the DMA global controller.

```c
int hosal_dma_init(void);
```

**Return value**: `0` success, `EIO` failure

---

### `hosal_dma_chan_request`

Request a DMA channel.

```c
hosal_dma_chan_t hosal_dma_chan_request(int flag);
```

| Parameter | Description |
|------|------|
| `flag` | Request flag (usually set to 0) |

**Return value**: Returns channel number (>=0) on success, negative number on failure

---

### `hosal_dma_chan_release`

Release a DMA channel.

```c
int hosal_dma_chan_release(hosal_dma_chan_t chan);
```

| Parameter | Description |
|------|------|
| `chan` | DMA channel number |

**Return value**: `0` success, `EIO` failure

---

### `hosal_dma_chan_start`

Start DMA transfer.

```c
int hosal_dma_chan_start(hosal_dma_chan_t chan);
```

---

### `hosal_dma_chan_stop`

Stop DMA transfer.

```c
int hosal_dma_chan_stop(hosal_dma_chan_t chan);
```

---

### `hosal_dma_irq_callback_set`

Set DMA transfer completion/error interrupt callback.

```c
int hosal_dma_irq_callback_set(hosal_dma_chan_t chan,
                               hosal_dma_irq_t pfn,
                               void *p_arg);
```

---

### `hosal_dma_finalize`

Release the DMA global controller.

```c
int hosal_dma_finalize(void);
```

## Usage Example

```c
#include "hal_dma.h"

// Global DMA initialization (usually called once during system initialization)
hosal_dma_init();

// Request a channel
hosal_dma_chan_t chan = hosal_dma_chan_request(0);
if (chan < 0) {
    // Request failed
}

// Set interrupt callback
hosal_dma_irq_callback_set(chan, my_dma_callback, NULL);

// Start transfer (actual transfer is triggered by peripheral driver)
hosal_dma_chan_start(chan);

// Stop after transfer completes
hosal_dma_chan_stop(chan);

// Release the channel
hosal_dma_chan_release(chan);
```
