# Flash API 参考

> 来源文件：`components/platform/hosal/include/hosal_flash.h`

## 宏定义

```c
#define HOSAL_FLASH_FLAG_ADDR_0     0       // 使用分区表地址 0
#define HOSAL_FLASH_FLAG_ADDR_1     (1<<0)  // 使用分区表地址 1
#define HOSAL_FLASH_FLAG_BUSADDR    (1<<1)   // 使用总线物理地址
```

## 类型定义

### `hosal_logic_partition_t` — Flash 分区信息结构

```c
typedef struct {
    const char  *partition_description; // 分区名称
    uint32_t     partition_start_addr; // 分区起始地址
    uint32_t     partition_length;     // 分区长度（字节）
    uint32_t     partition_options;    // 选项
} hosal_logic_partition_t;
```

### `hosal_flash_dev_t` — Flash 设备句柄

```c
typedef struct hosal_flash_dev {
    void *flash_dev;
} hosal_flash_dev_t;
```

> 通过 `hosal_flash_open()` 获取，不允许直接访问内部成员。

## 函数接口

### `hosal_flash_open`

打开一个 Flash 分区，获取设备句柄。

```c
hosal_flash_dev_t *hosal_flash_open(const char *name, unsigned int flags);
```

| 参数 | 说明 |
|------|------|
| `name` | 分区名称字符串，如 `"app"`、`"wifi"`、`"boot"` 等 |
| `flags` | 地址标志：`HOSAL_FLASH_FLAG_ADDR_0`、`HOSAL_FLASH_FLAG_ADDR_1`、`HOSAL_FLASH_FLAG_BUSADDR` |

**返回值**：成功返回设备句柄，失败返回 `NULL`

> 具体分区表定义在 SDK 的 `partition.csv` 文件中。

---

### `hosal_flash_info_get`

获取分区信息。

```c
int hosal_flash_info_get(hosal_flash_dev_t *p_dev,
                         hosal_logic_partition_t *partition);
```

| 参数 | 说明 |
|------|------|
| `p_dev` | `hosal_flash_open` 返回的设备句柄 |
| `partition` | 输出参数，存储分区信息 |

---

### `hosal_flash_erase`

擦除分区。

```c
int hosal_flash_erase(hosal_flash_dev_t *p_dev,
                       uint32_t off_set,
                       uint32_t size);
```

| 参数 | 说明 |
|------|------|
| `p_dev` | 设备句柄 |
| `off_set` | 分区内偏移（字节） |
| `size` | 擦除大小（字节） |

> 擦除按扇区进行，`size` 会被对齐到扇区边界。

---

### `hosal_flash_write`

写入 Flash（不自动擦除）。

```c
int hosal_flash_write(hosal_flash_dev_t *p_dev,
                      uint32_t *off_set,
                      const void *in_buf,
                      uint32_t in_buf_size);
```

| 参数 | 说明 |
|------|------|
| `p_dev` | 设备句柄 |
| `off_set` | 输入输出参数：写入起始位置，返回最后未写入地址 |
| `in_buf` | 数据缓冲区 |
| `in_buf_size` | 写入字节数 |

> 写入前必须确保目标区域已为 `0xFF`，否则数据出错。推荐使用 `hosal_flash_erase_write`。

---

### `hosal_flash_erase_write`

擦除并写入（常用）。

```c
int hosal_flash_erase_write(hosal_flash_dev_t *p_dev,
                            uint32_t *off_set,
                            const void *in_buf,
                            uint32_t in_buf_size);
```

| 参数 | 说明 |
|------|------|
| `off_set` | 输入输出参数：起始位置，返回最后未写入地址 |

---

### `hosal_flash_read`

读取 Flash。

```c
int hosal_flash_read(hosal_flash_dev_t *p_dev,
                     uint32_t *off_set,
                     void *out_buf,
                     uint32_t out_buf_size);
```

---

### `hosal_flash_raw_read`

直接读取 Flash 物理地址。

```c
int hosal_flash_raw_read(void *buffer, uint32_t address, uint32_t length);
```

---

### `hosal_flash_raw_write`

直接写入 Flash 物理地址（需先擦除）。

```c
int hosal_flash_raw_write(void *buffer, uint32_t address, uint32_t length);
```

---

### `hosal_flash_raw_erase`

直接擦除 Flash 物理地址。

```c
int hosal_flash_raw_erase(uint32_t start_addr, uint32_t length);
```

---

### `hosal_flash_close`

关闭 Flash 分区，释放设备句柄。

```c
int hosal_flash_close(hosal_flash_dev_t *p_dev);
```

## 使用示例

```c
#include "hal_flash.h"

#define FLASH_ADDR  0x1A0000
#define FLASH_SIZE  0x1000

// 打开 app 分区
hosal_flash_dev_t *flash = hosal_flash_open("app", HOSAL_FLASH_FLAG_ADDR_0);
if (flash == NULL) {
    printf("Flash open failed\r\n");
    return;
}

// 读取分区信息
hosal_logic_partition_t info;
hosal_flash_info_get(flash, &info);
printf("Partition: start=0x%x, size=0x%x\r\n",
       info.partition_start_addr, info.partition_length);

// 擦除 + 写入（安全方式）
uint8_t data[256] = {0x12, 0x34, 0x56, 0x78};
uint32_t offset = 0;
int ret = hosal_flash_erase_write(flash, &offset, data, sizeof(data));
if (ret != 0) {
    printf("Write failed\r\n");
}

// 读取
uint8_t read_buf[256];
offset = 0;
hosal_flash_read(flash, &offset, read_buf, sizeof(read_buf));

// 关闭
hosal_flash_close(flash);
```
