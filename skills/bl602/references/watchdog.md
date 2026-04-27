# Watchdog API Reference

> Source file: `components/platform/hosal/include/hosal_wdg.h`

## Type Definitions

### `hosal_wdg_config_t` — Watchdog Configuration Structure

```c
typedef struct {
    uint32_t timeout;  // Watchdog timeout (milliseconds)
} hosal_wdg_config_t;
```

### `hosal_wdg_dev_t` — Watchdog Device Structure

```c
typedef struct {
    uint8_t       port;
    hosal_wdg_config_t  config;
    void         *priv;
} hosal_wdg_dev_t;
```

## Function Interface

### `hosal_wdg_init`

Initializes the watchdog.

```c
int hosal_wdg_init(hosal_wdg_dev_t *wdg);
```

---

### `hosal_wdg_reload`

Feeds the watchdog (reloads the counter to prevent reset).

```c
void hosal_wdg_reload(hosal_wdg_dev_t *wdg);
```

---

### `hosal_wdg_finalize`

Releases the watchdog.

```c
int hosal_wdg_finalize(hosal_wdg_dev_t *wdg);
```

## Usage Example

```c
#include "hal_wdg.h"

hosal_wdg_dev_t wdg = {
    .port = 0,
    .config = {
        .timeout = 3000,  // 3 second timeout
    }
};

hosal_wdg_init(&wdg);

// In the main loop, periodically feed the watchdog
while (1) {
    hosal_wdg_reload(&wdg);  // Feed the watchdog
    // Business logic
    vTaskDelay(pdMS_TO_TICKS(500));
}
```
