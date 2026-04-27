# Flash API Reference

> Source file: `components/platform/hosal/include/hosal_flash.h`

## Macro Definitions

```c
#define HOSAL_FLASH_FLAG_ADDR_0     0       // Use partition table address 0
#define HOSAL_FLASH_FLAG_ADDR_1     (1<<0)  // Use partition table address 1
#define HOSAL_FLASH_FLAG_BUSADDR    (1<<1)   // Use bus physical address
```

## Type Definitions

### `hosal_logic_partition_t` — Flash Partition Information Structure

```c
typedef struct {
    const char  *partition_description; // Partition name
    uint32_t     partition_start_addr; // Partition start address
    uint32_t     partition_length;      // Partition length (bytes)
    uint32_t     partition_options;     // Options
} hosal_logic_partition_t;
```

### `hosal_flash_dev_t` — Flash Device Handle

```c
typedef struct hosal_flash_dev {
    void *flash_dev;
} hosal_flash_dev_t;
```

> Obtained via `hosal_flash_open()`. Do not access internal members directly.

## Function Interface

### `hosal_flash_open`

Opens a Flash partition and obtains a device handle.

```c
hosal_flash_dev_t *hosal_flash_open(const char *name, unsigned int flags);
```

| Parameter | Description |
|-----------|-------------|
| `name` | Partition name string, such as `"app"`, `"wifi"`, `"boot"`, etc. |
| `flags` | Address flags: `HOSAL_FLASH_FLAG_ADDR_0`, `HOSAL_FLASH_FLAG_ADDR_1`, `HOSAL_FLASH_FLAG_BUSADDR` |

**Return value**: Returns device handle on success, `NULL` on failure

> The specific partition table definition is in the SDK's `partition.csv` file.

---

### `hosal_flash_info_get`

Gets partition information.

```c
int hosal_flash_info_get(hosal_flash_dev_t *p_dev,
                         hosal_logic_partition_t *partition);
```

| Parameter | Description |
|-----------|-------------|
| `p_dev` | Device handle returned by `hosal_flash_open` |
| `partition` | Output parameter, stores partition information |

---

### `hosal_flash_erase`

Erases a partition.

```c
int hosal_flash_erase(hosal_flash_dev_t *p_dev,
                       uint32_t off_set,
                       uint32_t size);
```

| Parameter | Description |
|-----------|-------------|
| `p_dev` | Device handle |
| `off_set` | Offset within partition (bytes) |
| `size` | Size to erase (bytes) |

> Erasing is performed by sector. `size` will be aligned to sector boundaries.

---

### `hosal_flash_write`

Writes to Flash (does not auto-erase).

```c
int hosal_flash_write(hosal_flash_dev_t *p_dev,
                      uint32_t *off_set,
                      const void *in_buf,
                      uint32_t in_buf_size);
```

| Parameter | Description |
|-----------|-------------|
| `p_dev` | Device handle |
| `off_set` | Input/output parameter: write start position, returns last unwritten address |
| `in_buf` | Data buffer |
| `in_buf_size` | Number of bytes to write |

> Target area must be `0xFF` before writing, otherwise data will be corrupted. It is recommended to use `hosal_flash_erase_write`.

---

### `hosal_flash_erase_write`

Erases and writes (common usage).

```c
int hosal_flash_erase_write(hosal_flash_dev_t *p_dev,
                            uint32_t *off_set,
                            const void *in_buf,
                            uint32_t in_buf_size);
```

| Parameter | Description |
|-----------|-------------|
| `off_set` | Input/output parameter: start position, returns last unwritten address |

---

### `hosal_flash_read`

Reads Flash.

```c
int hosal_flash_read(hosal_flash_dev_t *p_dev,
                     uint32_t *off_set,
                     void *out_buf,
                     uint32_t out_buf_size);
```

---

### `hosal_flash_raw_read`

Directly reads Flash physical address.

```c
int hosal_flash_raw_read(void *buffer, uint32_t address, uint32_t length);
```

---

### `hosal_flash_raw_write`

Directly writes to Flash physical address (requires prior erase).

```c
int hosal_flash_raw_write(void *buffer, uint32_t address, uint32_t length);
```

---

### `hosal_flash_raw_erase`

Directly erases Flash physical address.

```c
int hosal_flash_raw_erase(uint32_t start_addr, uint32_t length);
```

---

### `hosal_flash_close`

Closes Flash partition and releases device handle.

```c
int hosal_flash_close(hosal_flash_dev_t *p_dev);
```

## Usage Example

```c
#include "hal_flash.h"

#define FLASH_ADDR  0x1A0000
#define FLASH_SIZE  0x1000

// Open app partition
hosal_flash_dev_t *flash = hosal_flash_open("app", HOSAL_FLASH_FLAG_ADDR_0);
if (flash == NULL) {
    printf("Flash open failed\r\n");
    return;
}

// Read partition info
hosal_logic_partition_t info;
hosal_flash_info_get(flash, &info);
printf("Partition: start=0x%x, size=0x%x\r\n",
       info.partition_start_addr, info.partition_length);

// Erase + write (safe method)
uint8_t data[256] = {0x12, 0x34, 0x56, 0x78};
uint32_t offset = 0;
int ret = hosal_flash_erase_write(flash, &offset, data, sizeof(data));
if (ret != 0) {
    printf("Write failed\r\n");
}

// Read
uint8_t read_buf[256];
offset = 0;
hosal_flash_read(flash, &offset, read_buf, sizeof(read_buf));

// Close
hosal_flash_close(flash);
```
