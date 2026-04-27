# GPIO API Reference

> Source file: `components/platform/hosal/include/hosal_gpio.h`

## Type Definitions

### `hosal_gpio_config_t` — GPIO Mode

```c
typedef enum {
    ANALOG_MODE,               // Analog mode (used as function pin)
    INPUT_PULL_UP,             // Input with pull-up (button connects to ground)
    INPUT_PULL_DOWN,           // Input with pull-down (button connects to power)
    INPUT_HIGH_IMPEDANCE,      // High impedance input (must be driven)
    OUTPUT_PUSH_PULL,          // Push-pull output (LED, etc.)
    OUTPUT_OPEN_DRAIN_NO_PULL, // Open-drain output (no pull-up)
    OUTPUT_OPEN_DRAIN_PULL_UP, // Open-drain output (internal pull-up)
    OUTPUT_OPEN_DRAIN_AF,      // Open-drain alternate function
    OUTPUT_PUSH_PULL_AF,       // Push-pull alternate function
} hosal_gpio_config_t;
```

### `hosal_gpio_irq_trigger_t` — Interrupt Trigger Type

```c
typedef enum {
    HOSAL_IRQ_TRIG_NEG_PULSE,  // Falling edge pulse trigger
    HOSAL_IRQ_TRIG_POS_PULSE,  // Rising edge pulse trigger
    HOSAL_IRQ_TRIG_NEG_LEVEL,  // Falling edge level trigger (32k 3T)
    HOSAL_IRQ_TRIG_POS_LEVEL,   // Rising edge level trigger (32k 3T)
} hosal_gpio_irq_trigger_t;
```

### `hosal_gpio_irq_handler_t` — Interrupt Callback Function Type

```c
typedef void (*hosal_gpio_irq_handler_t)(void *arg);
```

### `hosal_gpio_dev_t` — GPIO Device Structure

```c
typedef struct {
    uint8_t        port;         // GPIO port
    hosal_gpio_config_t  config; // GPIO configuration mode
    void          *priv;         // Private data
} hosal_gpio_dev_t;
```

## Function Interface

### `hosal_gpio_init`

Initializes a GPIO pin.

```c
int hosal_gpio_init(hosal_gpio_dev_t *gpio);
```

| Parameter | Description |
|-----------|-------------|
| `gpio` | GPIO device structure pointer |

**Return value**: `0` success, `EIO` failure

---

### `hosal_gpio_output_set`

Sets GPIO output level.

```c
int hosal_gpio_output_set(hosal_gpio_dev_t *gpio, uint8_t value);
```

| Parameter | Description |
|-----------|-------------|
| `gpio` | GPIO device |
| `value` | `0` = output low, `>0` = output high |

**Return value**: `0` success, `EIO` failure

---

### `hosal_gpio_input_get`

Reads GPIO input level.

```c
int hosal_gpio_input_get(hosal_gpio_dev_t *gpio, uint8_t *value);
```

| Parameter | Description |
|-----------|-------------|
| `gpio` | GPIO device |
| `value` | Output parameter, stores the read level value |

**Return value**: `0` success, `EIO` failure

---

### `hosal_gpio_irq_set`

Configures GPIO interrupt.

```c
int hosal_gpio_irq_set(hosal_gpio_dev_t *gpio,
                        hosal_gpio_irq_trigger_t trigger,
                        hosal_gpio_irq_handler_t handler,
                        void *arg);
```

| Parameter | Description |
|-----------|-------------|
| `gpio` | GPIO device |
| `trigger` | Trigger type |
| `handler` | Interrupt callback function |
| `arg` | Argument passed to the callback |

**Return value**: `0` success, `EIO` failure

---

### `hosal_gpio_irq_mask`

Masks or enables GPIO interrupt.

```c
int hosal_gpio_irq_mask(hosal_gpio_dev_t *gpio, uint8_t mask);
```

| Parameter | Description |
|-----------|-------------|
| `gpio` | GPIO device |
| `mask` | `0` = enable interrupt, `1` = mask interrupt |

**Return value**: `0` success, `EIO` failure

---

### `hosal_gpio_finalize`

Releases GPIO.

```c
int hosal_gpio_finalize(hosal_gpio_dev_t *gpio);
```

| Parameter | Description |
|-----------|-------------|
| `gpio` | GPIO device |

**Return value**: `0` success, `EIO` failure

## Usage Example

```c
#include "hal_gpio.h"

hosal_gpio_dev_t led = {
    .port = 0,
    .config = OUTPUT_PUSH_PULL,
};

// Initialize as push-pull output
hosal_gpio_init(&led);

// Output high level (turn on LED)
hosal_gpio_output_set(&led, 1);

// Input mode + interrupt
hosal_gpio_dev_t btn = {
    .port = 1,
    .config = INPUT_PULL_UP,
};
hosal_gpio_init(&btn);
hosal_gpio_irq_set(&btn, HOSAL_IRQ_TRIG_NEG_PULSE, my_handler, NULL);
```
