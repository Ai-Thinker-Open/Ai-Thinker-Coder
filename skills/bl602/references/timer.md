# Timer API 参考

> 来源文件：`components/platform/hosal/include/hosal_timer.h`

## 宏定义

```c
#define TIMER_RELOAD_PERIODIC 1  // 周期重复定时
#define TIMER_RELOAD_ONCE     2  // 单次定时
```

## 类型定义

### `hosal_timer_cb_t` — 定时器回调函数类型

```c
typedef void (*hosal_timer_cb_t)(void *arg);
```

### `hosal_timer_config_t` — 定时器配置结构

```c
typedef struct {
    uint32_t          period;      // 定时周期（微秒 us）
    uint8_t           reload_mode; // 重复模式：TIMER_RELOAD_PERIODIC / TIMER_RELOAD_ONCE
    hosal_timer_cb_t  cb;          // 定时回调函数
    void              *arg;         // 回调参数
} hosal_timer_config_t;
```

### `hosal_timer_dev_t` — 定时器设备结构

```c
typedef struct {
    int8_t                port;   // 定时器端口号
    hosal_timer_config_t  config;
    void                  *priv;
} hosal_timer_dev_t;
```

## 函数接口

### `hosal_timer_init`

初始化定时器。

```c
int hosal_timer_init(hosal_timer_dev_t *tim);
```

---

### `hosal_timer_start`

启动定时器。

```c
int hosal_timer_start(hosal_timer_dev_t *tim);
```

---

### `hosal_timer_stop`

停止定时器。

```c
void hosal_timer_stop(hosal_timer_dev_t *tim);
```

---

### `hosal_timer_finalize`

释放定时器。

```c
int hosal_timer_finalize(hosal_timer_dev_t *tim);
```

## 使用示例

```c
#include "hal_timer.h"

static void timer_callback(void *arg)
{
    printf("Timer expired!\r\n");
    // 处理定时事件
}

hosal_timer_dev_t tim0 = {
    .port = 0,
    .config = {
        .period = 1000000,          // 1 秒（1000000 us）
        .reload_mode = TIMER_RELOAD_PERIODIC,  // 周期重复
        .cb = timer_callback,
        .arg = NULL,
    }
};

hosal_timer_init(&tim0);
hosal_timer_start(&tim0);

// 需要停止时
hosal_timer_stop(&tim0);

// 需要释放时
hosal_timer_finalize(&tim0);
```
