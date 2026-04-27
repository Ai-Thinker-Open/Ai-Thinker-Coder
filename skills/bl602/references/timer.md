# BL602 Timer Register-Level API Reference

**Source file:** `components/platform/soc/bl602/bl602_std/bl602_std/Device/Bouffalo/BL602/Peripherals/timer_reg.h`

**Base Address:** `0x4000A500`

---

## Register Overview

| Offset | Register Name | Description |
|--------|--------------|-------------|
| `0x00` | TCCR | Timer Clock Control Register |
| `0x10` | TMR2_0 | Timer 2 Match Register 0 |
| `0x14` | TMR2_1 | Timer 2 Match Register 1 |
| `0x18` | TMR2_2 | Timer 2 Match Register 2 |
| `0x1C` | TMR3_0 | Timer 3 Match Register 0 |
| `0x20` | TMR3_1 | Timer 3 Match Register 1 |
| `0x24` | TMR3_2 | Timer 3 Match Register 2 |
| `0x2C` | TCR2 | Timer 2 Control Register |
| `0x30` | TCR3 | Timer 3 Control Register |
| `0x38` | TMSR2 | Timer 2 Match Status Register |
| `0x3C` | TMSR3 | Timer 3 Match Status Register |
| `0x44` | TIER2 | Timer 2 Interrupt Enable Register |
| `0x48` | TIER3 | Timer 3 Interrupt Enable Register |
| `0x50` | TPLVR2 | Timer 2 PWM Load Value Register |
| `0x54` | TPLVR3 | Timer 3 PWM Load Value Register |
| `0x5C` | TPLCR2 | Timer 2 PWM Load Control Register |
| `0x60` | TPLCR3 | Timer 3 PWM Load Control Register |
| `0x64` | WMER | Watchdog Mode Enable Register |
| `0x68` | WMR | Watchdog Match Register |
| `0x6C` | WVR | Watchdog Value Register |
| `0x70` | WSR | Watchdog Status Register |
| `0x78` | TICR2 | Timer 2 Interrupt Clear Register |
| `0x7C` | TICR3 | Timer 3 Interrupt Clear Register |
| `0x80` | WICR | Watchdog Interrupt Clear Register |
| `0x84` | TCER | Timer Enable Register |
| `0x88` | TCMR | Timer Mode Register |
| `0x90` | TILR2 | Timer 2 Interrupt Level Register |
| `0x94` | TILR3 | Timer 3 Interrupt Level Register |
| `0x98` | WCR | Watchdog Control Register |
| `0x9C` | WFAR | Watchdog Future Address Register |
| `0xA0` | WSAR | Watchdog Status Address Register |
| `0xA8` | TCVWR2 | Timer 2 Counter Write Register |
| `0xAC` | TCVWR3 | Timer 3 Counter Write Register |
| `0xB4` | TCVSYN2 | Timer 2 Counter Read/Write Sync Register |
| `0xB8` | TCVSYN3 | Timer 3 Counter Read/Write Sync Register |
| `0xBC` | TCDR | Timer Clock Divider Register |

---

## Key Register Fields

### TCCR - Timer Clock Control Register (Offset 0x00)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| CS_1 | [3:2] | R/W | Clock source for Timer 1 |
| CS_2 | [6:5] | R/W | Clock source for Timer 2 |
| CS_WDT | [9:8] | R/W | Clock source for Watchdog Timer |

### TMR2_x / TMR3_x - Timer Match Registers (Offsets 0x10-0x24)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| TMR | [31:0] | R/W | 32-bit match value for respective timer |

### TCR2 / TCR3 - Timer Control Registers (Offsets 0x2C, 0x30)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| TMR | [31:0] | R/W | 32-bit timer counter value |

### TMSR2 / TMSR3 - Timer Match Status Registers (Offsets 0x38, 0x3C)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| TMSR_0 | [0] | R/W | Timer match status for channel 0 |
| TMSR_1 | [1] | R/W | Timer match status for channel 1 |
| TMSR_2 | [2] | R/W | Timer match status for channel 2 |

### TIER2 / TIER3 - Timer Interrupt Enable Registers (Offsets 0x44, 0x48)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| TIER_0 | [0] | R/W | Interrupt enable for channel 0 |
| TIER_1 | [1] | R/W | Interrupt enable for channel 1 |
| TIER_2 | [2] | R/W | Interrupt enable for channel 2 |

### TPLCR2 / TPLCR3 - Timer PWM Load Control Registers (Offsets 0x5C, 0x60)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| TPLCR | [1:0] | R/W | PWM load trigger control |

### WMER - Watchdog Mode Enable Register (Offset 0x64)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| WE | [0] | R/W | Watchdog enable |
| WRIE | [1] | R/W | Watchdog reset interrupt enable |

### WMR - Watchdog Match Register (Offset 0x68)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| WMR | [15:0] | R/W | Watchdog match value |

### WVR - Watchdog Value Register (Offset 0x6C)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| WVR | [15:0] | R/W | Watchdog counter value |

### WSR - Watchdog Status Register (Offset 0x70)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| WTS | [0] | R/W | Watchdog timer status |

### TCER - Timer Enable Register (Offset 0x84)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| TIMER2_EN | [1] | R/W | Timer 2 enable |
| TIMER3_EN | [2] | R/W | Timer 3 enable |

### TCMR - Timer Mode Register (Offset 0x88)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| TIMER2_MODE | [1] | R/W | Timer 2 mode (0=free-run, 1=user-defined) |
| TIMER3_MODE | [2] | R/W | Timer 3 mode (0=free-run, 1=user-defined) |

### TILR2 / TILR3 - Timer Interrupt Level Registers (Offsets 0x90, 0x94)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| TILR_0 | [0] | R/W | Interrupt level for channel 0 |
| TILR_1 | [1] | R/W | Interrupt level for channel 1 |
| TILR_2 | [2] | R/W | Interrupt level for channel 2 |

### TCDR - Timer Clock Divider Register (Offset 0xBC)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| TCDR2 | [15:8] | R/W | Timer 2 clock pre-divider |
| TCDR3 | [23:16] | R/W | Timer 3 clock pre-divider |
| WCDR | [31:24] | R/W | Watchdog clock pre-divider |

---

## Register-Level Programming Example

### Timer 2 Periodic Interrupt Example

```c
#include "bl602.h"
#include "bl602_timer.h"

#define TIMER_BASE 0x4000A500

/* Timer interrupt handler */
void timer2_irq_handler(void)
{
    volatile timer_reg_t *timer = (volatile timer_reg_t *)TIMER_BASE;

    /* Check match status for channel 0 */
    if (timer->TMSR2.BF.tmsr_0) {
        /* Clear interrupt */
        timer->TICR2.WORD = (1 << 0);

        /* Handle timer event */
        // ...
    }
}

/* Initialize Timer 2 for periodic interrupt */
void timer2_init(uint32_t period_ticks)
{
    volatile timer_reg_t *timer = (volatile timer_reg_t *)TIMER_BASE;

    /* Stop timer first */
    timer->TCER.BF.timer2_en = 0;

    /* Set match value for channel 0 */
    timer->TMR2_0.WORD = period_ticks;

    /* Set clock divider (e.g., divide by 1) */
    timer->TCDR.WORD &= ~TIMER_TCDR2_MSK;
    timer->TCDR.BF.tcdr2 = 1;

    /* Configure clock source (e.g., APB clock) */
    timer->TCCR.BF.cs_2 = 0x1;  /* Timer 2 clock source */

    /* Enable interrupt for channel 0 */
    timer->TIER2.BF.tier_0 = 1;

    /* Set interrupt level */
    timer->TILR2.BF.tilr_0 = 1;

    /* Configure for periodic mode via TCMR if needed */
    timer->TCMR.BF.timer2_mode = 1;

    /* Enable timer */
    timer->TCER.BF.timer2_en = 1;
}

/* Stop Timer 2 */
void timer2_stop(void)
{
    volatile timer_reg_t *timer = (volatile timer_reg_t *)TIMER_BASE;
    timer->TCER.BF.timer2_en = 0;
}
```

### Watchdog Timer Example

```c
/* Initialize Watchdog Timer */
void wdt_init(uint16_t timeout_ticks)
{
    volatile timer_reg_t *timer = (volatile timer_reg_t *)TIMER_BASE;

    /* Set watchdog match value */
    timer->WMR.BF.wmr = timeout_ticks;

    /* Set clock divider */
    timer->TCDR.BF.wcdr = 0xFF;  /* Max divider */

    /* Configure watchdog mode */
    timer->WMER.BF.we = 1;    /* Enable watchdog */
    timer->WMER.BF.wrie = 1;  /* Enable reset interrupt */

    /* Start watchdog */
    timer->WSR.BF.wts = 1;
}

/* Feed watchdog */
void wdt_feed(void)
{
    volatile timer_reg_t *timer = (volatile timer_reg_t *)TIMER_BASE;
    timer->WSR.BF.wts = 1;
}
```

---

## HOSAL API

**Header:** `components/platform/hosal/include/hosal_timer.h`

### Data Types

```c
#define TIMER_RELOAD_PERIODIC 1  /* Timer auto-reloads on expiry */
#define TIMER_RELOAD_ONCE     2  /* Timer expires once, needs manual reload */

typedef void (*hosal_timer_cb_t)(void *arg);

typedef struct {
    uint32_t          period;      /* Timer period in microseconds */
    uint8_t           reload_mode; /* RELOAD_PERIODIC or RELOAD_ONCE */
    hosal_timer_cb_t  cb;          /* Callback function on expiry */
    void              *arg;        /* Callback argument */
} hosal_timer_config_t;

typedef struct {
    int8_t                port;
    hosal_timer_config_t  config;
    void                  *priv;
} hosal_timer_dev_t;
```

### API Functions

| Function | Description |
|----------|-------------|
| `int hosal_timer_init(hosal_timer_dev_t *tim)` | Initialize timer with configuration |
| `int hosal_timer_start(hosal_timer_dev_t *tim)` | Start the timer |
| `void hosal_timer_stop(hosal_timer_dev_t *tim)` | Stop the timer |
| `int hosal_timer_finalize(hosal_timer_dev_t *tim)` | De-initialize timer |

### HOSAL Usage Example

```c
#include "hosal_timer.h"

hosal_timer_dev_t timer_dev;
volatile uint32_t timer_count = 0;

void timer_callback(void *arg)
{
    timer_count++;
    // Handle timer expiry
}

void app_timer_init(void)
{
    timer_dev.port = 0;  /* Timer port 0 (Timer 2) */
    timer_dev.config.period = 1000000;  /* 1 second period */
    timer_dev.config.reload_mode = TIMER_RELOAD_PERIODIC;
    timer_dev.config.cb = timer_callback;
    timer_dev.config.arg = NULL;

    hosal_timer_init(&timer_dev);
    hosal_timer_start(&timer_dev);
}

void app_timer_stop(void)
{
    hosal_timer_stop(&timer_dev);
    hosal_timer_finalize(&timer_dev);
}
```
