# PWM API Reference

> Source file: `components/platform/hosal/include/hosal_pwm.h`

## Type Definitions

### `hosal_pwm_config_t` — PWM Configuration Structure

```c
typedef struct {
    uint8_t    pin;        // PWM pin
    uint32_t   duty_cycle; // Duty cycle, range 0~10000 (corresponding to 0%~100%)
    uint32_t   freq;       // Frequency Hz, max 40MHz
} hosal_pwm_config_t;
```

> Note: `duty_cycle` uses ten-thousandths as unit, 5000 = 50% duty cycle.

### `hosal_pwm_dev_t` — PWM Device Structure

```c
typedef struct {
    uint8_t       port;         // PWM port
    hosal_pwm_config_t  config;
    void         *priv;
} hosal_pwm_dev_t;
```

## Function Interface

### `hosal_pwm_init`

Initialize PWM.

```c
int hosal_pwm_init(hosal_pwm_dev_t *pwm);
```

---

### `hosal_pwm_start`

Start PWM output.

```c
int hosal_pwm_start(hosal_pwm_dev_t *pwm);
```

---

### `hosal_pwm_stop`

Stop PWM output.

```c
int hosal_pwm_stop(hosal_pwm_dev_t *pwm);
```

---

### `hosal_pwm_para_chg`

Dynamically change PWM parameters (frequency + duty cycle updated simultaneously).

```c
int hosal_pwm_para_chg(hosal_pwm_dev_t *pwm, hosal_pwm_config_t para);
```

---

### `hosal_pwm_freq_set`

Set PWM frequency individually.

```c
int hosal_pwm_freq_set(hosal_pwm_dev_t *pwm, uint32_t freq);
```

| Parameter | Description |
|-----------|-------------|
| `pwm` | PWM device |
| `freq` | Frequency Hz (0~40M) |

---

### `hosal_pwm_freq_get`

Get current PWM frequency.

```c
int hosal_pwm_freq_get(hosal_pwm_dev_t *pwm, uint32_t *p_freq);
```

---

### `hosal_pwm_duty_set`

Set PWM duty cycle individually.

```c
int hosal_pwm_duty_set(hosal_pwm_dev_t *pwm, uint32_t duty);
```

| Parameter | Description |
|-----------|-------------|
| `duty` | Duty cycle, range 0~10000 (5000 = 50%) |

---

### `hosal_pwm_duty_get`

Get current PWM duty cycle.

```c
int hosal_pwm_duty_get(hosal_pwm_dev_t *pwm, uint32_t *p_duty);
```

---

### `hosal_pwm_finalize`

Release PWM.

```c
int hosal_pwm_finalize(hosal_pwm_dev_t *pwm);
```

## Usage Example

```c
#include "hal_pwm.h"

// Initialize: 10kHz, 50% duty cycle
hosal_pwm_dev_t pwm0 = {
    .port = 0,
    .config = {
        .pin = 10,
        .freq = 10000,        // 10kHz
        .duty_cycle = 5000,   // 50% (5000/10000)
    }
};

hosal_pwm_init(&pwm0);
hosal_pwm_start(&pwm0);

// Dynamic adjustment: change to 80% duty cycle
hosal_pwm_duty_set(&pwm0, 8000);

// Dynamic adjustment: change to 1kHz frequency
hosal_pwm_freq_set(&pwm0, 1000);

// Change both frequency and duty cycle simultaneously
hosal_pwm_config_t new_para = {
    .pin = 10,
    .freq = 5000,
    .duty_cycle = 2500,  // 25%
};
hosal_pwm_para_chg(&pwm0, new_para);

// Stop
hosal_pwm_stop(&pwm0);
```
