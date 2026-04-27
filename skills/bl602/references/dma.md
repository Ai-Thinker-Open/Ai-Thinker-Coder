# DMA API 参考

> 来源文件：`components/platform/hosal/include/hosal_dma.h`

## 宏定义

```c
#define HOSAL_DMA_INT_TRANS_COMPLETE 0  // 传输完成中断
#define HOSAL_DMA_INT_TRANS_ERROR    1  // 传输错误中断
```

## 类型定义

### `hosal_dma_irq_t` — DMA 中断回调类型

```c
typedef void (*hosal_dma_irq_t)(void *p_arg, uint32_t flag);
```

- `flag` = `0` (`HOSAL_DMA_INT_TRANS_COMPLETE`)：传输完成
- `flag` = `1` (`HOSAL_DMA_INT_TRANS_ERROR`)：传输错误

### `hosal_dma_chan_t` — DMA 通道号

```c
typedef int hosal_dma_chan_t;
```

通道号为整数（如 0、1、2 等）。

### `hosal_dma_dev_t` — DMA 设备结构

```c
typedef struct hosal_dma_dev {
    int max_chans;                    // 最大通道数
    struct hosal_dma_chan *used_chan; // 已用通道数组
    void *priv;
} hosal_dma_dev_t;
```

## 函数接口

### `hosal_dma_init`

初始化 DMA 全局控制器。

```c
int hosal_dma_init(void);
```

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_dma_chan_request`

请求一个 DMA 通道。

```c
hosal_dma_chan_t hosal_dma_chan_request(int flag);
```

| 参数 | 说明 |
|------|------|
| `flag` | 请求标志（通常填 0） |

**返回值**：成功返回通道号（>=0），失败返回负数

---

### `hosal_dma_chan_release`

释放 DMA 通道。

```c
int hosal_dma_chan_release(hosal_dma_chan_t chan);
```

| 参数 | 说明 |
|------|------|
| `chan` | DMA 通道号 |

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_dma_chan_start`

启动 DMA 传输。

```c
int hosal_dma_chan_start(hosal_dma_chan_t chan);
```

---

### `hosal_dma_chan_stop`

停止 DMA 传输。

```c
int hosal_dma_chan_stop(hosal_dma_chan_t chan);
```

---

### `hosal_dma_irq_callback_set`

设置 DMA 传输完成/错误中断回调。

```c
int hosal_dma_irq_callback_set(hosal_dma_chan_t chan,
                               hosal_dma_irq_t pfn,
                               void *p_arg);
```

---

### `hosal_dma_finalize`

释放 DMA 全局控制器。

```c
int hosal_dma_finalize(void);
```

## 使用示例

```c
#include "hal_dma.h"

// 全局 DMA 初始化（通常在系统初始化时调用一次）
hosal_dma_init();

// 请求通道
hosal_dma_chan_t chan = hosal_dma_chan_request(0);
if (chan < 0) {
    // 请求失败
}

// 设置中断回调
hosal_dma_irq_callback_set(chan, my_dma_callback, NULL);

// 启动传输（具体传输由外设驱动触发）
hosal_dma_chan_start(chan);

// 传输完成后停止
hosal_dma_chan_stop(chan);

// 释放通道
hosal_dma_chan_release(chan);
```
