# BL602 Watchdog Register-Level API Reference

**Source File:** `components/platform/soc/bl602/bl602_std/bl602_std/Device/Bouffalo/BL602/Peripherals/timer_reg.h`  
**HOSAL Header:** `components/platform/hosal/include/hosal_wdg.h`  
**Base Address:** `TIMER_BASE = 0x4000A500`  
**Interrupt:** `TIMER_WDT_IRQn` (IRQ 38)  
**Driver Source:** `components/platform/soc/bl602/bl602_std/bl602_std/StdDriver/Src/bl602_timer.c`

---

## Register Overview

The Watchdog Timer is part of the Timer peripheral. It provides system reset on timeout to recover from software failures.

| Offset | Register Name | Description |
|--------|--------------|-------------|
| `0x00` | `TIMER_WDT_CURRENT` | Current watchdog value |
| `0x04` | `TIMER_WDT_RELOAD` | Watchdog reload value |
| `0x08` | `TIMER_WDT_CMD` | Watchdog command register |
| `0x0C` | `TIMER_WDT_INT_CFG` | Watchdog interrupt configuration |
| `0x10` | `TIMER_WDT_RESET_MASK` | Reset mask register |
| `0x14` | `TIMER_WDT_LOCK` | Lock register |
| `0x18` | `TIMER_WDT_INT_STATUS` | Watchdog interrupt status |

---

## Key Register Fields

### TIMER_WDT_CURRENT (Offset 0x00)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `WDT_CURRENT` | [31:0] | Current Value | Current watchdog counter value |

### TIMER_WDT_RELOAD (Offset 0x04)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `WDT_RELOAD` | [31:0] | Reload Value | Value loaded on restart/timeout |

### TIMER_WDT_CMD (Offset 0x08)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `WDT_EN` | [0] | Watchdog Enable | 1=enable watchdog |
| `WDT_RST` | [1] | Reset Enable | 1=reset system on timeout |
| `WDT_INT_EN` | [2] | Interrupt Enable | 1=enable pre-timeout interrupt |
| `WDT_CLK_PRESCALE` | [15:8] | Clock Prescale | Clock divider: tick = PCLK / (prescale+1) |
| `WDT_MODE` | [16] | Mode | 0=free-running, 1=periodic |
| `WDT_RLD_EN` | [17] | Auto Reload | 1=auto-reload after interrupt |

### TIMER_WDT_INT_CFG (Offset 0x0C)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `WDT_INT_PERIOD` | [15:0] | Interrupt Period | Pre-timeout interrupt period |
| `WDT_RST_PERIOD` | [31:16] | Reset Period | Timeout period for reset |

### TIMER_WDT_RESET_MASK (Offset 0x10)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `WDT_RST_MASK` | [0] | Reset Mask | 1=mask watchdog reset (for debug) |

### TIMER_WDT_INT_STATUS (Offset 0x18)

| Field | Bits | Name | Description |
|-------|------|------|-------------|
| `WDT_INT_CLR` | [0] | Interrupt Clear | Write 1 to clear interrupt |
| `WDT_TIMEOUT` | [1] | Timeout Flag | 1=watchdog timed out |
| `WDT_INT_RAW` | [2] | Raw Interrupt | Pre-timeout interrupt raw status |

---

## Register-Level Programming Example

```c
#include "bl602.h"
#include "timer_reg.h"

#define TIMER_BASE  ((uint32_t)0x4000A500)

/* Initialize and start watchdog */
void WDT_Start(uint32_t timeout_ms)
{
    uint32_t tick_rate;
    uint32_t reload_val;
    uint32_t tmpVal;

    /* Calculate reload value
     * Timer runs at PCLK / (prescale + 1)
     * Typical PCLK = 80MHz
     * Prescale 79 gives 1MHz tick rate (1us per tick)
     */
    tmpVal = BL_RD_REG(TIMER_BASE, TIMER_WDT_CMD);
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, WDT_CLK_PRESCALE, 79); /* 1MHz tick */
    tmpVal = BL_SET_REG_BIT(tmpVal, WDT_RST);    /* enable reset */
    tmpVal = BL_SET_REG_BIT(tmpVal, WDT_EN);     /* enable WDT */
    BL_WR_REG(TIMER_BASE, TIMER_WDT_CMD, tmpVal);

    /* Set reload value (timeout in ticks) */
    reload_val = timeout_ms * 1000;  /* convert ms to us (ticks) */
    BL_WR_REG(TIMER_BASE, TIMER_WDT_RELOAD, reload_val);
}

/* Initialize watchdog with pre-timeout interrupt */
void WDT_Start_With_Pretimeout(uint32_t pretimeout_ms, uint32_t full_timeout_ms)
{
    uint32_t tmpVal;

    /* Configure prescaler for 1MHz tick */
    tmpVal = 0;
    tmpVal = BL_SET_REG_BITS_VAL(tmpVal, WDT_CLK_PRESCALE, 79);
    tmpVal = BL_SET_REG_BIT(tmpVal, WDT_RST);
    tmpVal = BL_SET_REG_BIT(tmpVal, WDT_INT_EN);  /* enable interrupt */
    tmpVal = BL_SET_REG_BIT(tmpVal, WDT_MODE);    /* periodic mode */
    BL_WR_REG(TIMER_BASE, TIMER_WDT_CMD, tmpVal);

    /* Set interrupt period and reset period */
    BL_WR_REG(TIMER_BASE, TIMER_WDT_INT_CFG,
              (pretimeout_ms * 1000) | ((full_timeout_ms * 1000) << 16));

    /* Set reload value */
    BL_WR_REG(TIMER_BASE, TIMER_WDT_RELOAD, full_timeout_ms * 1000);
}

/* Feed the watchdog (reload counter) */
void WDT_Feed(void)
{
    uint32_t tmpVal;

    /* Write current value to reload register to restart */
    tmpVal = BL_RD_REG(TIMER_BASE, TIMER_WDT_CURRENT);
    BL_WR_REG(TIMER_BASE, TIMER_WDT_RELOAD, tmpVal);
}

/* Stop watchdog */
void WDT_Stop(void)
{
    uint32_t tmpVal;

    tmpVal = BL_RD_REG(TIMER_BASE, TIMER_WDT_CMD);
    tmpVal = BL_CLR_REG_BIT(tmpVal, WDT_EN);
    BL_WR_REG(TIMER_BASE, TIMER_WDT_CMD, tmpVal);
}

/* Clear watchdog interrupt */
void WDT_Clear_Interrupt(void)
{
    uint32_t tmpVal;

    tmpVal = BL_RD_REG(TIMER_BASE, TIMER_WDT_INT_STATUS);
    tmpVal = BL_SET_REG_BIT(tmpVal, WDT_INT_CLR);
    BL_WR_REG(TIMER_BASE, TIMER_WDT_INT_STATUS, tmpVal);
}

/* Watchdog interrupt handler example */
void WDT_IRQHandler(void)
{
    uint32_t status;

    status = BL_RD_REG(TIMER_BASE, TIMER_WDT_INT_STATUS);

    if (status & (1 << 2)) {  /* pre-timeout interrupt */
        /* Save state, log warning, or feed watchdog */
        WDT_Feed();
    }

    if (status & (1 << 1)) {  /* timeout occurred */
        /* This will lead to system reset if WDT_RST is enabled */
    }

    /* Clear interrupt */
    BL_WR_REG(TIMER_BASE, TIMER_WDT_INT_STATUS, 1);
}
```

---

## HOSAL API

**Header:** `components/platform/hosal/include/hosal_wdg.h`

```c
/* HOSAL Watchdog device configuration */
typedef struct {
    uint32_t timeout; /*!< Watchdog timeout in ms */
} hosal_wdg_config_t;

/* HOSAL Watchdog device structure */
typedef struct {
    uint8_t       port;   /* wdg port (typically 0) */
    hosal_wdg_config_t config; /* wdg config */
    void         *priv;   /* private data */
} hosal_wdg_dev_t;

/**
 * @brief Initialize the on-board CPU hardware watchdog
 * @param wdg  watchdog device pointer
 * @return 0 on success, other on failure
 */
int hosal_wdg_init(hosal_wdg_dev_t *wdg);

/**
 * @brief Reload watchdog counter (feed the dog)
 * @param wdg  watchdog device pointer
 */
void hosal_wdg_reload(hosal_wdg_dev_t *wdg);

/**
 * @brief Deinitialize watchdog device
 * @param wdg  watchdog device pointer
 * @return 0 on success, other on failure
 */
int hosal_wdg_finalize(hosal_wdg_dev_t *wdg);
```

### HOSAL Watchdog Usage Example

```c
#include "hosal_wdg.h"

hosal_wdg_dev_t wdg;

void wdg_example(void)
{
    int ret;

    /* Initialize watchdog with 5 second timeout */
    wdg.port = 0;
    wdg.config.timeout = 5000;  /* 5 seconds */

    ret = hosal_wdg_init(&wdg);
    if (ret != 0) {
        printf("Watchdog init failed\r\n");
        return;
    }

    /* Main loop */
    while (1) {
        /* Do work... */

        /* Feed the watchdog */
        hosal_wdg_reload(&wdg);

        /* Small delay */
        for (volatile int i = 0; i < 100000; i++);
    }
}

/* Using interrupt-based pre-timeout warning */
void wdg_with_callback_example(void)
{
    hosal_wdg_dev_t wdg;

    wdg.port = 0;
    wdg.config.timeout = 10000;  /* 10 second timeout */

    hosal_wdg_init(&wdg);

    while (1) {
        /* Application logic */

        /* For safety, feed before timeout */
        hosal_wdg_reload(&wdg);
    }
}
```

### Notes

- The watchdog uses a 32-bit counter running from the APB clock
- The maximum timeout depends on prescaler and clock frequency
- At 80MHz with prescale 79 (1MHz tick), maximum timeout is ~71 minutes
- The watchdog reset is a chip-level reset, not just CPU reset
- During debugging, mask the reset to prevent unwanted resets (WDT_RST_MASK)
- The pre-timeout interrupt (if enabled) fires before the actual reset
- Always feed the watchdog before it times out to prevent system reset
- If the watchdog times out and reset is enabled, the entire chip will reset
