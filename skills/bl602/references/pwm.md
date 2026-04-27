# BL602 PWM Register-Level API Reference

**Source file:** `components/platform/soc/bl602/bl602_std/bl602_std/Device/Bouffalo/BL602/Peripherals/pwm_reg.h`

**Base Address:** `0x4000A400`

---

## Register Overview

| Offset | Register Name | Description |
|--------|--------------|-------------|
| `0x00` | PWM_INT_CONFIG | Global PWM Interrupt Configuration |
| `0x20` | PWM0_CLKDIV | PWM Channel 0 Clock Divider |
| `0x24` | PWM0_THRE1 | PWM Channel 0 Threshold 1 (Fall Edge) |
| `0x28` | PWM0_THRE2 | PWM Channel 0 Threshold 2 (Rise Edge) |
| `0x2C` | PWM0_PERIOD | PWM Channel 0 Period |
| `0x30` | PWM0_CONFIG | PWM Channel 0 Configuration |
| `0x34` | PWM0_INTERRUPT | PWM Channel 0 Interrupt Control |
| `0x40` | PWM1_CLKDIV | PWM Channel 1 Clock Divider |
| `0x44` | PWM1_THRE1 | PWM Channel 1 Threshold 1 |
| `0x48` | PWM1_THRE2 | PWM Channel 1 Threshold 2 |
| `0x4C` | PWM1_PERIOD | PWM Channel 1 Period |
| `0x50` | PWM1_CONFIG | PWM Channel 1 Configuration |
| `0x54` | PWM1_INTERRUPT | PWM Channel 1 Interrupt Control |
| `0x60` | PWM2_CLKDIV | PWM Channel 2 Clock Divider |
| `0x64` | PWM2_THRE1 | PWM Channel 2 Threshold 1 |
| `0x68` | PWM2_THRE2 | PWM Channel 2 Threshold 2 |
| `0x6C` | PWM2_PERIOD | PWM Channel 2 Period |
| `0x70` | PWM2_CONFIG | PWM Channel 2 Configuration |
| `0x74` | PWM2_INTERRUPT | PWM Channel 2 Interrupt Control |
| `0x80` | PWM3_CLKDIV | PWM Channel 3 Clock Divider |
| `0x84` | PWM3_THRE1 | PWM Channel 3 Threshold 1 |
| `0x88` | PWM3_THRE2 | PWM Channel 3 Threshold 2 |
| `0x8C` | PWM3_PERIOD | PWM Channel 3 Period |
| `0x90` | PWM3_CONFIG | PWM Channel 3 Configuration |
| `0x94` | PWM3_INTERRUPT | PWM Channel 3 Interrupt Control |
| `0xA0` | PWM4_CLKDIV | PWM Channel 4 Clock Divider |
| `0xA4` | PWM4_THRE1 | PWM Channel 4 Threshold 1 |
| `0xA8` | PWM4_THRE2 | PWM Channel 4 Threshold 2 |
| `0xAC` | PWM4_PERIOD | PWM Channel 4 Period |
| `0xB0` | PWM4_CONFIG | PWM Channel 4 Configuration |
| `0xB4` | PWM4_INTERRUPT | PWM Channel 4 Interrupt Control |

---

## Key Register Fields

### PWM_INT_CONFIG - Global Interrupt Config (Offset 0x00)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| PWM_INTERRUPT_STS | [5:0] | R | PWM interrupt status (one bit per channel) |
| PWM_INT_CLEAR | [13:8] | W | Interrupt clear bits (write 1 to clear) |

### PWMx_CLKDIV - Clock Divider (Offsets 0x20, 0x40, 0x60, 0x80, 0xA0)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| PWM_CLK_DIV | [15:0] | R/W | Clock divider value (0 = bypass) |

### PWMx_THRE1 - Threshold 1 (Offsets 0x24, 0x44, 0x64, 0x84, 0xA4)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| PWM_THRE1 | [15:0] | R/W | Fall edge threshold (duty cycle off point) |

### PWMx_THRE2 - Threshold 2 (Offsets 0x28, 0x48, 0x68, 0x88, 0xA8)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| PWM_THRE2 | [15:0] | R/W | Rise edge threshold (duty cycle on point) |

### PWMx_PERIOD - Period (Offsets 0x2C, 0x4C, 0x6C, 0x8C, 0xAC)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| PWM_PERIOD | [15:0] | R/W | PWM period count (clock ticks per period) |

### PWMx_CONFIG - Configuration (Offsets 0x30, 0x50, 0x70, 0x90, 0xB0)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| REG_CLK_SEL | [1:0] | R/W | Clock source select (0=PCLK, 1=32K, 2=32M, 3=external) |
| OUT_INV | [2] | R/W | Output inversion enable |
| STOP_MODE | [3] | R/W | Stop mode (0=pause, 1=freeze) |
| SW_FORCE_VAL | [4] | R/W | Software force output value |
| SW_MODE | [5] | R/W | Software force mode enable |
| STOP_EN | [6] | R/W | Stop enable (allows PWM to stop) |
| STS_TOP | [7] | R/W | Status of PWM at top (period start) |

### PWMx_INTERRUPT - Interrupt Control (Offsets 0x34, 0x54, 0x74, 0x94, 0xB4)

| Field | Bits | Access | Description |
|-------|------|--------|-------------|
| INT_PERIOD_CNT | [15:0] | R/W | Interrupt period count (periods before interrupt) |
| INT_ENABLE | [16] | R/W | Interrupt enable |

---

## Register-Level Programming Example

### PWM Output Example (Channel 0)

```c
#include "bl602.h"
#include "bl602_pwm.h"

#define PWM_BASE 0x4000A400

/* Configure PWM channel 0 for 1kHz, 50% duty cycle */
void pwm0_init(uint32_t freq_hz, uint32_t duty_percent)
{
    volatile pwm_reg_t *pwm = (volatile pwm_reg_t *)PWM_BASE;
    uint32_t pclk_freq = 80000000;  /* Example: 80MHz APB clock */
    uint32_t period, thre1, thre2;

    /* Calculate divider: PWM freq = PCLK / (divider * period) */
    /* For simplicity, use divider = 1 */
    period = pclk_freq / freq_hz;

    /* Duty cycle thresholds */
    thre2 = (period * duty_percent) / 100;  /* Rise edge */
    thre1 = period;                          /* Fall edge at period end */

    /* Stop PWM first */
    pwm->pwm0_config.BF.stop_en = 1;
    pwm->pwm0_config.BF.sw_mode = 0;

    /* Set clock divider (divisor = 0 means divide by 1) */
    pwm->pwm0_clkdiv.WORD = 0;

    /* Set period */
    pwm->pwm0_period.WORD = period;

    /* Set thresholds */
    pwm->pwm0_thre2.WORD = thre2;
    pwm->pwm0_thre1.WORD = thre1;

    /* Configure clock source: 0 = PCLK */
    pwm->pwm0_config.BF.reg_clk_sel = 0;

    /* Enable output, no inversion */
    pwm->pwm0_config.BF.out_inv = 0;

    /* Set stop mode: 0 = pause on stop */
    pwm->pwm0_config.BF.stop_mode = 0;

    /* Enable PWM */
    pwm->pwm0_config.BF.stop_en = 1;
}

/* Start PWM channel 0 */
void pwm0_start(void)
{
    volatile pwm_reg_t *pwm = (volatile pwm_reg_t *)PWM_BASE;
    pwm->pwm0_config.BF.sw_mode = 1;
    pwm->pwm0_config.BF.sw_force_val = 1;
    pwm->pwm0_config.BF.sw_mode = 0;
}

/* Stop PWM channel 0 */
void pwm0_stop(void)
{
    volatile pwm_reg_t *pwm = (volatile pwm_reg_t *)PWM_BASE;
    pwm->pwm0_config.BF.sw_mode = 1;
    pwm->pwm0_config.BF.sw_force_val = 0;
}

/* Change PWM duty cycle at runtime */
void pwm0_set_duty(uint32_t duty_percent)
{
    volatile pwm_reg_t *pwm = (volatile pwm_reg_t *)PWM_BASE;
    uint32_t period = pwm->pwm0_period.WORD;
    uint32_t thre2 = (period * duty_percent) / 100;

    pwm->pwm0_thre2.WORD = thre2;
}
```

### PWM Interrupt Example

```c
/* Configure PWM channel 0 with periodic interrupt */
void pwm0_interrupt_init(void)
{
    volatile pwm_reg_t *pwm = (volatile pwm_reg_t *)PWM_BASE;

    /* Set period */
    pwm->pwm0_period.WORD = 1000;

    /* Interrupt every 100 periods */
    pwm->pwm0_interrupt.BF.int_period_cnt = 100;

    /* Enable interrupt */
    pwm->pwm0_interrupt.BF.int_enable = 1;
}

/* PWM interrupt handler */
void pwm0_irq_handler(void)
{
    volatile pwm_reg_t *pwm = (volatile pwm_reg_t *)PWM_BASE;

    /* Check if channel 0 triggered interrupt */
    if (pwm->pwm_int_config.BF.pwm_interrupt_sts & (1 << 0)) {
        /* Clear interrupt */
        pwm->pwm_int_config.BF.pwm_int_clear = (1 << 0);

        /* Handle PWM interrupt */
        // ...
    }
}
```

---

## HOSAL API

**Header:** `components/platform/hosal/include/hosal_pwm.h`

### Data Types

```c
typedef struct {
    uint8_t    pin;           /* PWM pin */
    uint32_t   duty_cycle;    /* Duty cycle: 0 ~ 10000 (0 ~ 100%) */
    uint32_t   freq;          /* Frequency: 0 ~ 40M Hz */
} hosal_pwm_config_t;

typedef struct {
    uint8_t       port;        /* PWM port (0-4) */
    hosal_pwm_config_t config; /* PWM configuration */
    void          *priv;
} hosal_pwm_dev_t;
```

### API Functions

| Function | Description |
|----------|-------------|
| `int hosal_pwm_init(hosal_pwm_dev_t *pwm)` | Initialize PWM with configuration |
| `int hosal_pwm_start(hosal_pwm_dev_t *pwm)` | Start PWM output |
| `int hosal_pwm_stop(hosal_pwm_dev_t *pwm)` | Stop PWM output |
| `int hosal_pwm_para_chg(hosal_pwm_dev_t *pwm, hosal_pwm_config_t para)` | Change PWM parameters |
| `int hosal_pwm_freq_set(hosal_pwm_dev_t *pwm, uint32_t freq)` | Set PWM frequency |
| `int hosal_pwm_freq_get(hosal_pwm_dev_t *pwm, uint32_t *p_freq)` | Get PWM frequency |
| `int hosal_pwm_duty_set(hosal_pwm_dev_t *pwm, uint32_t duty)` | Set PWM duty cycle |
| `int hosal_pwm_duty_get(hosal_pwm_dev_t *pwm, uint32_t *p_duty)` | Get PWM duty cycle |
| `int hosal_pwm_finalize(hosal_pwm_dev_t *pwm)` | De-initialize PWM |

### HOSAL Usage Example

```c
#include "hosal_pwm.h"

hosal_pwm_dev_t pwm_dev;

void app_pwm_init(void)
{
    pwm_dev.port = 0;  /* PWM channel 0 */
    pwm_dev.config.pin = 0;  /* Pin 0 */
    pwm_dev.config.freq = 1000;       /* 1 kHz */
    pwm_dev.config.duty_cycle = 5000;  /* 50% duty cycle */

    hosal_pwm_init(&pwm_dev);
    hosal_pwm_start(&pwm_dev);
}

void app_pwm_change_duty(uint32_t new_duty)
{
    /* Duty cycle: 0-10000 represents 0-100% */
    hosal_pwm_duty_set(&pwm_dev, new_duty);
}

void app_pwm_stop(void)
{
    hosal_pwm_stop(&pwm_dev);
    hosal_pwm_finalize(&pwm_dev);
}
```
