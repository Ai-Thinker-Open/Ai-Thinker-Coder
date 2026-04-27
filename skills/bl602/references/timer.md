# Timer API Reference

> Source file: `components/platform/hosal/include/hosal_timer.h`

## Macros

```c
#define TIMER_RELOAD_PERIODIC 1  // Periodic reload (continuous)
#define TIMER_RELOAD_ONCE     2  // One-shot timer
```

## Type Definitions

### `hosal_timer_cb_t` — Timer Callback Function Type

```c
typedef void (*hosal_timer_cb_t)(void *arg);
```

### `hosal_timer_config_t` — Timer Configuration Structure

```c
typedef struct {
    uint32_t          period;      // Timer period (microseconds)
    uint8_t           reload_mode; // Reload mode: TIMER_RELOAD_PERIODIC / TIMER_RELOAD_ONCE
    hosal_timer_cb_t  cb;          // Timer callback function
    void              *arg;         // Callback argument
} hosal_timer_config_t;
```

### `hosal_timer_dev_t` — Timer Device Structure

```c
typedef struct {
    int8_t                port;   // Timer port number
    hosal_timer_config_t  config;
    void                  *priv;
} hosal_timer_dev_t;
```

## Function API

### `hosal_timer_init`

Initialize timer.

```c
int hosal_timer_init(hosal_timer_dev_t *tim);
```

---

### `hosal_timer_start`

Start timer.

```c
int hosal_timer_start(hosal_timer_dev_t *tim);
```

---

### `hosal_timer_stop`

Stop timer.

```c
void hosal_timer_stop(hosal_timer_dev_t *tim);
```

---

### `hosal_timer_finalize`

Finalize timer.

```c
int hosal_timer_finalize(hosal_timer_dev_t *tim);
```

## Usage Example

```c
#include "hal_timer.h"

static void timer_callback(void *arg)
{
    printf("Timer expired!\r\n");
    // Handle timer event
}

hosal_timer_dev_t tim0 = {
    .port = 0,
    .config = {
        .period = 1000000,          // 1 second (1000000 us)
        .reload_mode = TIMER_RELOAD_PERIODIC,  // Periodic reload
        .cb = timer_callback,
        .arg = NULL,
    }
};

hosal_timer_init(&tim0);
hosal_timer_start(&tim0);

// When you need to stop
hosal_timer_stop(&tim0);

// When you need to finalize
hosal_timer_finalize(&tim0);
```
