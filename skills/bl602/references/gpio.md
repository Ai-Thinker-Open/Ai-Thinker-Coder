# GPIO API Reference (Register-Level)

> HOSAL Header: `components/platform/hosal/include/hosal_gpio.h`  
> Register Header: `components/platform/soc/bl602/bl602_std/.../StdDriver/Inc/bl602_gpio.h`  
> GLB Register Map: `Device/Bouffalo/BL602/Peripherals/glb_reg.h`  
> GPIO pin count: 23 (GPIO 0–22)

---

## Register Overview

GPIO is part of the **GLB (Global Bus)** peripheral at `0x40000000`. There is **no separate GPIO peripheral base address** — all GPIO registers are embedded within the GLB register map.

Each GPIO pin has one **32-bit configuration register** at `GLB_BASE + 0x100 + (pin × 4)`.

| GPIO Pin | Config Register | Address |
|---|---|---|
| GPIO 0 | `GLB_GPIO_CFGCTL0` | `0x40000100` |
| GPIO 1 | `GLB_GPIO_CFGCTL1` | `0x40000104` |
| GPIO 2 | `GLB_GPIO_CFGCTL2` | `0x40000108` |
| ... | ... | ... |
| GPIO 22 | `GLB_GPIO_CFGCTL22` | `0x40000158` |

---

## Register Map (GPIO Config Registers)

| Offset | Register | Description |
|---|---|---|
| `0x100` + n×4 | `GLB_GPIO_CFGCTLn` | GPIO 0–22 configuration (n = pin number) |
| `0x400` | `GPIO_INT_CTRL` | GPIO interrupt control (shared) |
| `0x404`` | `GPIO_INT_STAT` | GPIO interrupt status |
| `0x408` | `GPIO_INT_SET` | GPIO interrupt trigger set |
| `0x40C` | `GPIO_INT_CLEAR` | GPIO interrupt clear |

---

## Field Description

### `GLB_GPIO_CFGCTLn` (offset = `0x100 + n×4`, n = 0–22)

Each GPIO pin uses 16 bits in its configuration register:

| Field | Bits | Access | Description |
|---|---|---|---|
| `IE` | [0] | RW | Input enable (1=enable input path) |
| `SMT` | [1] | RW | Schmitt trigger (1=enable) |
| `DRV` | [3:2] | RW | Drive strength: 0=5mA, 1=10mA, 2=15mA, 3=20mA |
| `PU` | [4] | RW | Pull-up enable |
| `PD` | [5] | RW | Pull-down enable |
| `FUNC_SEL` | [12:8] | RW | Pin function select (see pin function table) |
| `REAL_FUNC_SEL` | [16:12] | RO | Actual function selected (hardware mirror) |

> **Note**: Pull-up and pull-down should not both be enabled simultaneously. Use `PU=1,PD=0` for pull-up, `PU=0,PD=1` for pull-down, `PU=0,PD=0` for no pull.

---

## Pin Function Selection (`FUNC_SEL` Encoding)

| Value | Function |
|---|---|
| `0` | GPIO (software GPIO mode) |
| `1` | SDIO |
| `2` | SPI Flash |
| `4` | SPI |
| `6` | I2C |
| `7` | UART |
| `8` | PWM |
| `9` | External PA |
| `10` | Analog |
| `11` | SWGPIO (software GPIO alternate) |
| `14` | JTAG |

---

## Output Level Registers

| Register | Address | Description |
|---|---|---|
| `GPIO_OUTPUT` | `0x40000180` | Output data register. Bit[n] = GPIO n output level |
| `GPIO_OUTPUT_ENABLE` | `0x40000184` | Output enable. Bit[n]=1 enables output driver for GPIO n |
| `GPIO_INPUT` | `0x40000188` | Input data register. Read-only, reflects actual pin state |

---

## Interrupt Registers

### `GPIO_INT_CTRL` (0x40000400)

| Field | Bits | Description |
|---|---|---|
| `INT_CTRL` | [0] | Global GPIO interrupt enable |

### `GPIO_INT_STAT` (0x40000404)

| Field | Bits | Description |
|---|---|---|
| `INT_STAT` | [22:0] | One bit per GPIO pin. 1=interrupt pending for that pin |

### `GPIO_INT_SET` (0x40000408)

Write 1 to set interrupt flag for a GPIO pin.

### `GPIO_INT_CLEAR` (0x4000040C)

Write 1 to clear interrupt flag for a GPIO pin.

---

## Bit-Level Access Macros

The SDK defines standard bit manipulation macros in `bl602_glb.h`:

```c
// Read-modify-write a field (non-destructive)
reg = getreg32(addr);
reg = (reg & ~FIELD_MSK) | (FIELD_VAL << FIELD_POS);
putreg32(addr, reg);

// Or use the mask/pos macros directly
*(uint32_t *)addr = (*(uint32_t *)addr & ~GLB_REG_GPIO_0_FUNC_SEL_MSK)
                    | (func << GLB_REG_GPIO_0_FUNC_SEL_POS);
```

---

## Register-Level Programming Sequence

### Configure GPIO as Output (e.g., GPIO 5, push-pull, 10mA)

```c
#define GLB_BASE  0x40000000
#define GPIO_REG(pin)  (*(volatile uint32_t *)(GLB_BASE + 0x100 + (pin) * 4))
#define GPIO_OUT_EN    (*(volatile uint32_t *)(GLB_BASE + 0x184))

// Step 1: Configure pin function = GPIO (FUNC_SEL = 0)
GPIO_REG(5) = (0 << 8)   // FUNC_SEL = 0 (GPIO mode)
             | (1 << 2)   // DRV = 1 (10mA)
             | (0 << 1)   // SMT = 0
             | (0 << 0);  // IE = 0 (output mode, disable input)

// Step 2: Set output high (write to output register)
*(volatile uint32_t *)(GLB_BASE + 0x180) |= (1 << 5);

// Step 3: Enable output driver
GPIO_OUT_EN |= (1 << 5);
```

### Configure GPIO as Input (e.g., GPIO 12, pull-up, with interrupt)

```c
#define GLB_BASE      0x40000000
#define GPIO_REG(pin) (*(volatile uint32_t *)(GLB_BASE + 0x100 + (pin) * 4))
#define GPIO_OUT_EN   (*(volatile uint32_t *)(GLB_BASE + 0x184))
#define GPIO_INT_SET  (*(volatile uint32_t *)(GLB_BASE + 0x408))
#define GPIO_INT_CTRL (*(volatile uint32_t *)(GLB_BASE + 0x400))

// Configure GPIO 12 as input with pull-up
GPIO_REG(12) = (0 << 8)   // FUNC_SEL = 0 (GPIO)
             | (1 << 4)   // PU = 1 (pull-up enabled)
             | (0 << 5)   // PD = 0
             | (1 << 1)   // SMT = 1
             | (1 << 0);  // IE = 1 (input enabled)

// Disable output driver
GPIO_OUT_EN &= ~(1 << 12);

// Enable interrupt on GPIO 12
GPIO_INT_SET = (1 << 12);
GPIO_INT_CTRL |= 1;  // Global GPIO interrupt enable
```

### Toggle GPIO (toggle without read-modify-write)

```c
// Use XOR to toggle output state
*(volatile uint32_t *)(GLB_BASE + 0x180) ^= (1 << 5);  // Toggle GPIO 5
```

### Read Input Level

```c
uint32_t level = (*(volatile uint32_t *)(GLB_BASE + 0x188)) & (1 << 12);
// level != 0 means HIGH, level == 0 means LOW
```

---

## HOSAL API (Higher-Level Abstraction)

For application code, prefer the HOSAL API rather than direct register access.

```c
#include "hosal_gpio.h"

// Initialize GPIO
hosal_gpio_dev_t gpio5 = {
    .port = 5,
    .config = OUTPUT_PUSH_PULL,
};
hosal_gpio_init(&gpio5);

// Set output high
hosal_gpio_output_set(&gpio5, 1);

// Set output low
hosal_gpio_output_set(&gpio5, 0);

// Initialize GPIO input with interrupt
hosal_gpio_dev_t gpio12 = {
    .port = 12,
    .config = INPUT_PULL_UP,
};
hosal_gpio_init(&gpio12);
hosal_gpio_irq_init(&gpio12, HOSAL_IRQ_TRIG_POS_EDGE, my_isr, NULL);

// Read input
uint32_t val;
hosal_gpio_input_get(&gpio12, &val);
```

---

## Common Mistakes

1. **Forgetting to disable output driver** when configuring as input — the pin may fight the external signal.
2. **Both pull-up and pull-down enabled** — causes indeterminate state. Set `PU=0,PD=0` for high-impedance input.
3. **Writing `FUNC_SEL=0` without clearing first** — FUNC_SEL is 4 bits; you must write the complete value, not OR in a field.
4. **Output enable is in a separate register** (`0x184`) — setting the config register alone does not enable output. Write `GPIO_OUT_EN |= (1<<pin)`.
