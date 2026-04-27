# Flash 分区管理 (BL_MTD) API 参考

> 来源文件：`components/sys/blmtd/include/bl_mtd.h`  
> 统一管理 Flash 分区（PSM、FW、media 等）的读写擦除。

---

## 概述

BL_MTD（Memory Technology Device）是 Flash 分区抽象层，封装了对 Flash 各分区的统一访问接口，提供 open/read/write/erase 语义。

预定义分区名称：
- `BL_MTD_PARTITION_NAME_PSM` — PSM 持久化存储区
- `BL_MTD_PARTITION_NAME_FW_DEFAULT` — 固件区
- `BL_MTD_PARTITION_NAME_ROMFS` — ROMFS 多媒体区

---

## 头文件

```c
#include "bl_mtd.h"
```

---

## 类型定义

### `bl_mtd_handle_t`

分区操作句柄：

```c
typedef void *bl_mtd_handle_t;
```

### `bl_mtd_info_t`

分区信息结构体：

```c
typedef struct {
    char name[16];         // 分区名称
    unsigned int offset;   // Flash 偏移地址
    unsigned int size;     // 分区大小（字节）
    void *xip_addr;        // XIP 映射地址（只读）
} bl_mtd_info_t;
```

---

## 宏

### 打开标志

```c
#define BL_MTD_OPEN_FLAG_NONE     (0)       // 普通打开
#define BL_MTD_OPEN_FLAG_BACKUP   (1 << 0)  // 打开备份分区
#define BL_MTD_OPEN_FLAG_BUSADDR  (1 << 1)  // 使用总线地址（不用 XIP）
```

---

## 函数接口

### `bl_mtd_open`

打开分区。

```c
int bl_mtd_open(const char *name, bl_mtd_handle_t *handle, unsigned int flags);
```

| 参数 | 说明 |
|------|------|
| `name` | 分区名称 |
| `handle` | 输出句柄 |
| `flags` | 打开标志 |

**返回值**：0=成功

---

### `bl_mtd_close`

关闭分区。

```c
int bl_mtd_close(bl_mtd_handle_t handle);
```

---

### `bl_mtd_info`

获取分区信息。

```c
int bl_mtd_info(bl_mtd_handle_t handle, bl_mtd_info_t *info);
```

---

### `bl_mtd_erase`

擦除指定地址范围。

```c
int bl_mtd_erase(bl_mtd_handle_t handle, unsigned int addr,
                 unsigned int size);
```

| 参数 | 说明 |
|------|------|
| `addr` | 分区内偏移（从 0 开始） |
| `size` | 擦除大小 |

---

### `bl_mtd_erase_all`

擦除整个分区。

```c
int bl_mtd_erase_all(bl_mtd_handle_t handle);
```

---

### `bl_mtd_write`

写入数据（自动擦除）。

```c
int bl_mtd_write(bl_mtd_handle_t handle, unsigned int addr,
                 unsigned int size, const uint8_t *data);
```

---

### `bl_mtd_read`

读取数据。

```c
int bl_mtd_read(bl_mtd_handle_t handle, unsigned int addr,
                unsigned int size, uint8_t *data);
```

---

### `bl_mtd_size`

获取分区大小。

```c
int bl_mtd_size(bl_mtd_handle_t handle, unsigned int *size);
```

---

## 使用示例

### 读取 PSM 分区

```c
#include "bl_mtd.h"

int read_psm_calibration(void)
{
    bl_mtd_handle_t handle;
    int ret = bl_mtd_open(BL_MTD_PARTITION_NAME_PSM, &handle,
                           BL_MTD_OPEN_FLAG_NONE);
    if (ret != 0) {
        printf("Failed to open PSM\r\n");
        return -1;
    }

    bl_mtd_info_t info;
    bl_mtd_info(handle, &info);
    printf("PSM: offset=0x%x size=%u\r\n", info.offset, info.size);

    uint8_t data[32];
    ret = bl_mtd_read(handle, 0, sizeof(data), data);
    bl_mtd_close(handle);

    return ret;
}
```

### 读写固件配置区

```c
int save_config(uint8_t *config, size_t len)
{
    bl_mtd_handle_t handle;
    int ret = bl_mtd_open("FW", &handle, BL_MTD_OPEN_FLAG_NONE);
    if (ret != 0) return -1;

    // 擦除后写入
    ret = bl_mtd_erase(handle, 0, len);
    if (ret == 0) {
        ret = bl_mtd_write(handle, 0, len, config);
    }

    bl_mtd_close(handle);
    return ret;
}
```
