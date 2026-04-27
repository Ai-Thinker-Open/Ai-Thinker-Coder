# Watchdog API 参考

> 来源文件：`components/platform/hosal/include/hosal_wdg.h`

## 类型定义

### `hosal_wdg_config_t` — 看门狗配置结构

```c
typedef struct {
    uint32_t timeout;  // 看门狗超时时间（毫秒 ms）
} hosal_wdg_config_t;
```

### `hosal_wdg_dev_t` — 看门狗设备结构

```c
typedef struct {
    uint8_t       port;
    hosal_wdg_config_t  config;
    void         *priv;
} hosal_wdg_dev_t;
```

## 函数接口

### `hosal_wdg_init`

初始化看门狗。

```c
int hosal_wdg_init(hosal_wdg_dev_t *wdg);
```

---

### `hosal_wdg_reload`

喂狗（重载计数器，防止复位）。

```c
void hosal_wdg_reload(hosal_wdg_dev_t *wdg);
```

---

### `hosal_wdg_finalize`

释放看门狗。

```c
int hosal_wdg_finalize(hosal_wdg_dev_t *wdg);
```

## 使用示例

```c
#include "hal_wdg.h"

hosal_wdg_dev_t wdg = {
    .port = 0,
    .config = {
        .timeout = 3000,  // 3 秒超时
    }
};

hosal_wdg_init(&wdg);

// 主循环中定期喂狗
while (1) {
    hosal_wdg_reload(&wdg);  // 喂狗
    // 业务逻辑
    vTaskDelay(pdMS_TO_TICKS(500));
}
```
