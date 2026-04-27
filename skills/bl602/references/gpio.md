# GPIO API 参考

> 来源文件：`components/platform/hosal/include/hosal_gpio.h`

## 类型定义

### `hosal_gpio_config_t` — GPIO 模式

```c
typedef enum {
    ANALOG_MODE,               // 模拟模式（用作功能引脚）
    INPUT_PULL_UP,             // 输入上拉（按钮接地）
    INPUT_PULL_DOWN,           // 输入下拉（按钮接电源）
    INPUT_HIGH_IMPEDANCE,      // 高阻输入（必须被驱动）
    OUTPUT_PUSH_PULL,          // 推挽输出（LED 等）
    OUTPUT_OPEN_DRAIN_NO_PULL, // 开漏输出（无上拉）
    OUTPUT_OPEN_DRAIN_PULL_UP, // 开漏输出（内置上拉）
    OUTPUT_OPEN_DRAIN_AF,      // 开漏复用功能
    OUTPUT_PUSH_PULL_AF,       // 推挽复用功能
} hosal_gpio_config_t;
```

### `hosal_gpio_irq_trigger_t` — 中断触发类型

```c
typedef enum {
    HOSAL_IRQ_TRIG_NEG_PULSE,  // 下降沿脉冲触发
    HOSAL_IRQ_TRIG_POS_PULSE,  // 上升沿脉冲触发
    HOSAL_IRQ_TRIG_NEG_LEVEL,  // 下降沿电平触发（32k 3T）
    HOSAL_IRQ_TRIG_POS_LEVEL,   // 上升沿电平触发（32k 3T）
} hosal_gpio_irq_trigger_t;
```

### `hosal_gpio_irq_handler_t` — 中断回调函数类型

```c
typedef void (*hosal_gpio_irq_handler_t)(void *arg);
```

### `hosal_gpio_dev_t` — GPIO 设备结构

```c
typedef struct {
    uint8_t        port;         // GPIO 端口
    hosal_gpio_config_t  config; // GPIO 配置模式
    void          *priv;         // 私有数据
} hosal_gpio_dev_t;
```

## 函数接口

### `hosal_gpio_init`

初始化 GPIO 引脚。

```c
int hosal_gpio_init(hosal_gpio_dev_t *gpio);
```

| 参数 | 说明 |
|------|------|
| `gpio` | GPIO 设备结构体指针 |

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_gpio_output_set`

设置 GPIO 输出电平。

```c
int hosal_gpio_output_set(hosal_gpio_dev_t *gpio, uint8_t value);
```

| 参数 | 说明 |
|------|------|
| `gpio` | GPIO 设备 |
| `value` | `0` = 输出低，`>0` = 输出高 |

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_gpio_input_get`

读取 GPIO 输入电平。

```c
int hosal_gpio_input_get(hosal_gpio_dev_t *gpio, uint8_t *value);
```

| 参数 | 说明 |
|------|------|
| `gpio` | GPIO 设备 |
| `value` | 输出参数，存储读取的电平值 |

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_gpio_irq_set`

配置 GPIO 中断。

```c
int hosal_gpio_irq_set(hosal_gpio_dev_t *gpio,
                        hosal_gpio_irq_trigger_t trigger,
                        hosal_gpio_irq_handler_t handler,
                        void *arg);
```

| 参数 | 说明 |
|------|------|
| `gpio` | GPIO 设备 |
| `trigger` | 触发类型 |
| `handler` | 中断回调函数 |
| `arg` | 传递给回调的参数 |

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_gpio_irq_mask`

屏蔽或使能 GPIO 中断。

```c
int hosal_gpio_irq_mask(hosal_gpio_dev_t *gpio, uint8_t mask);
```

| 参数 | 说明 |
|------|------|
| `gpio` | GPIO 设备 |
| `mask` | `0` = 使能中断，`1` = 屏蔽中断 |

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_gpio_finalize`

释放 GPIO。

```c
int hosal_gpio_finalize(hosal_gpio_dev_t *gpio);
```

| 参数 | 说明 |
|------|------|
| `gpio` | GPIO 设备 |

**返回值**：`0` 成功，`EIO` 失败

## 使用示例

```c
#include "hal_gpio.h"

hosal_gpio_dev_t led = {
    .port = 0,
    .config = OUTPUT_PUSH_PULL,
};

// 初始化为推挽输出
hosal_gpio_init(&led);

// 输出高电平（点亮 LED）
hosal_gpio_output_set(&led, 1);

// 输入模式 + 中断
hosal_gpio_dev_t btn = {
    .port = 1,
    .config = INPUT_PULL_UP,
};
hosal_gpio_init(&btn);
hosal_gpio_irq_set(&btn, HOSAL_IRQ_TRIG_NEG_PULSE, my_handler, NULL);
```
