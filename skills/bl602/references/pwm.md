# PWM API 参考

> 来源文件：`components/platform/hosal/include/hosal_pwm.h`

## 类型定义

### `hosal_pwm_config_t` — PWM 配置结构

```c
typedef struct {
    uint8_t    pin;        // PWM 引脚
    uint32_t   duty_cycle; // 占空比，范围 0~10000（对应 0%~100%）
    uint32_t   freq;       // 频率 Hz，最大 40MHz
} hosal_pwm_config_t;
```

> 注意：`duty_cycle` 使用万分之一为单位，5000 = 50% 占空比。

### `hosal_pwm_dev_t` — PWM 设备结构

```c
typedef struct {
    uint8_t       port;         // PWM 端口
    hosal_pwm_config_t  config;
    void         *priv;
} hosal_pwm_dev_t;
```

## 函数接口

### `hosal_pwm_init`

初始化 PWM。

```c
int hosal_pwm_init(hosal_pwm_dev_t *pwm);
```

---

### `hosal_pwm_start`

启动 PWM 输出。

```c
int hosal_pwm_start(hosal_pwm_dev_t *pwm);
```

---

### `hosal_pwm_stop`

停止 PWM 输出。

```c
int hosal_pwm_stop(hosal_pwm_dev_t *pwm);
```

---

### `hosal_pwm_para_chg`

动态更改 PWM 参数（频率 + 占空比同时更新）。

```c
int hosal_pwm_para_chg(hosal_pwm_dev_t *pwm, hosal_pwm_config_t para);
```

---

### `hosal_pwm_freq_set`

单独设置 PWM 频率。

```c
int hosal_pwm_freq_set(hosal_pwm_dev_t *pwm, uint32_t freq);
```

| 参数 | 说明 |
|------|------|
| `pwm` | PWM 设备 |
| `freq` | 频率 Hz（0~40M） |

---

### `hosal_pwm_freq_get`

获取当前 PWM 频率。

```c
int hosal_pwm_freq_get(hosal_pwm_dev_t *pwm, uint32_t *p_freq);
```

---

### `hosal_pwm_duty_set`

单独设置 PWM 占空比。

```c
int hosal_pwm_duty_set(hosal_pwm_dev_t *pwm, uint32_t duty);
```

| 参数 | 说明 |
|------|------|
| `duty` | 占空比，范围 0~10000（5000 = 50%） |

---

### `hosal_pwm_duty_get`

获取当前 PWM 占空比。

```c
int hosal_pwm_duty_get(hosal_pwm_dev_t *pwm, uint32_t *p_duty);
```

---

### `hosal_pwm_finalize`

释放 PWM。

```c
int hosal_pwm_finalize(hosal_pwm_dev_t *pwm);
```

## 使用示例

```c
#include "hal_pwm.h"

// 初始化：10kHz，50% 占空比
hosal_pwm_dev_t pwm0 = {
    .port = 0,
    .config = {
        .pin = 10,
        .freq = 10000,        // 10kHz
        .duty_cycle = 5000,   // 50%（5000/10000）
    }
};

hosal_pwm_init(&pwm0);
hosal_pwm_start(&pwm0);

// 动态调整：改为 80% 占空比
hosal_pwm_duty_set(&pwm0, 8000);

// 动态调整：改为 1kHz 频率
hosal_pwm_freq_set(&pwm0, 1000);

// 同时更改频率和占空比
hosal_pwm_config_t new_para = {
    .pin = 10,
    .freq = 5000,
    .duty_cycle = 2500,  // 25%
};
hosal_pwm_para_chg(&pwm0, new_para);

// 停止
hosal_pwm_stop(&pwm0);
```
