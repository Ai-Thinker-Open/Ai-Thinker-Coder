# RNG API 参考

> 来源文件：`components/platform/hosal/include/hosal_rng.h`

## 函数接口

### `hosal_rng_init`

初始化随机数生成器。

```c
int hosal_rng_init(void);
```

**返回值**：`0` 成功，其他失败

> 调用 `hosal_random_num_read` 前必须先调用此函数进行初始化。

---

### `hosal_random_num_read`

读取随机数填充缓冲区。

```c
int hosal_random_num_read(void *buf, uint32_t bytes);
```

| 参数 | 说明 |
|------|------|
| `buf` | 有效内存缓冲区，随机数将填充到此内存 |
| `bytes` | 缓冲区长度（字节） |

**返回值**：`0` 成功，其他失败

## 使用示例

```c
#include "hal_rng.h"

// 初始化 RNG（通常在系统初始化时调用一次）
hosal_rng_init();

// 读取 8 字节随机数
uint8_t random_bytes[8];
int ret = hosal_random_num_read(random_bytes, 8);
if (ret == 0) {
    printf("Random: %02X%02X%02X%02X%02X%02X%02X%02X\r\n",
           random_bytes[0], random_bytes[1], random_bytes[2], random_bytes[3],
           random_bytes[4], random_bytes[5], random_bytes[6], random_bytes[7]);
}

// 生成随机数用于密钥、随机延迟、跳频等场景
uint32_t random_val;
hosal_random_num_read(&random_val, sizeof(random_val));
```

## 应用场景

| 场景 | 说明 |
|------|------|
| 密钥生成 | 加密通信的会话密钥 |
| 随机延迟 | 避免无线通信冲突的随机退避时间 |
| MAC 地址 | 生成随机 MAC 地址用于测试 |
| 跳频种子 | 跳频通信的伪随机序列种子 |
