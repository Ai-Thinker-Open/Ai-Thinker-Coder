# Efuse API 参考

> 来源文件：`components/platform/hosal/include/hosal_efuse.h`

> Efuse（电子熔丝）是 BL602 芯片内部一次性可编程存储区域，用于存放不可修改的硬件配置数据（如 MAC 地址、加密密钥、校准参数等）。读操作可多次，写操作通常只能执行一次，部分位为永久锁定。

## 函数接口

### `hosal_efuse_read`

从 Efuse 读取数据。

```c
int hosal_efuse_read(uint32_t addr, uint32_t *data, uint32_t len);
```

| 参数 | 说明 |
|------|------|
| `addr` | Efuse 地址（字节偏移） |
| `data` | 读取数据存储缓冲区 |
| `len` | 读取长度（字节），建议按 4 字节对齐 |

**返回值**：`0` 成功，`EIO` 失败

---

### `hosal_efuse_write`

向 Efuse 写入数据（通常一次性）。

```c
int hosal_efuse_write(uint32_t addr, uint32_t *data, uint32_t len);
```

| 参数 | 说明 |
|------|------|
| `addr` | Efuse 地址（字节偏移） |
| `data` | 待写入数据缓冲区 |
| `len` | 写入长度（字节），建议按 4 字节对齐 |

**返回值**：`0` 成功，`EIO` 失败

> **注意**：写入操作不可逆，写入后无法恢复。建议先读取确认当前值，确认为全 `0xFF`（空白）后再写入。

## 使用示例

```c
#include "hal_efuse.h"

uint32_t data[2] = {0};

// 读取 Efuse addr=0x10 处 8 字节数据
int ret = hosal_efuse_read(0x10, data, 8);
if (ret == 0) {
    printf("Efuse[0x10]: 0x%08X\r\n", data[0]);
}

// 写入 Efuse addr=0x20 处 4 字节数据（谨慎操作）
uint32_t write_data = 0x12345678;
ret = hosal_efuse_write(0x20, &write_data, 4);
if (ret == 0) {
    printf("Efuse write success\r\n");
}
```

## 常用 Efuse 地址（BL602）

| 地址 | 内容 | 说明 |
|------|------|------|
| 0x00 | Chip ID | 芯片批次号 |
| 0x10 | MAC Address[0] | Wi-Fi MAC 地址低 32 位 |
| 0x14 | MAC Address[1] | Wi-Fi MAC 地址高 16 位 + 其他 |
| 0x20 | Security Key | 密钥数据 |

> 不同固件版本地址定义可能不同，请参考官方 SDK 文档或 `bl_efuse.h`。
