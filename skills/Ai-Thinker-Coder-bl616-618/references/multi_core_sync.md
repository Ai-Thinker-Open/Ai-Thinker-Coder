# Multi-Core Sync API Reference (BL618DG)

> **Source:** `bouffalo_sdk/drivers/lhal/include/bflb_multi_core_sync.h`  
> **Implementation:** `bouffalo_sdk/drivers/lhal/src/bflb_multi_core_sync.c`  
> **依赖:** `drivers/lhal/include/hardware/ipc_reg.h`, `components/ipc/ipm.h`
>
> **⚠️ 芯片支持：** 多核同步 API 仅在 **BL618DG** 多核芯片上可用，且需要启用 `CONFIG_IPC` 配置。BL616 单核芯片无此外设。

## Overview

Multi-Core Sync 模块为 BL618DG 多核系统（AP + NP 核心）提供 Flash 操作期间的核间同步机制。当 AP 核心需要执行 Flash 擦除、写入或读取操作时，通过 IPC 同步机制先挂起 (Suspend) NP 核心，操作完成后再恢复 (Resume) NP 核心，确保 Flash 操作的原子性和数据一致性。

**主要特性：**
- Flash 擦除/写入/读取操作的 NP 核心同步保护
- IPC 同步协议：Suspend → 操作 → Resume
- 超时处理机制（3 秒 Suspend 等待，1 秒 Resume 等待）
- 系统复位时的 NP 核心安全挂起

**工作流程：**

```
AP 核                                   NP 核
  │                                       │
  ├─ IPC_SYNC_SUSPEND_CMD ──────────────►│  挂起 NP
  │  ◄────────────── IPC_SYNC_SUSPEND_ACK │
  │                                       │
  ├─ Flash Erase / Write / Read           │  NP 已暂停
  │                                       │
  ├─ IPC_SYNC_RESUME_CMD ───────────────►│  恢复 NP
  │  ◄────────────── IPC_SYNC_RESUME_ACK  │
  │                                       │
```

---

## API Functions

### bflb_flash_erase_mcs

多核安全的 Flash 擦除操作。

```c
int bflb_flash_erase_mcs(uint32_t erase_addr, uint32_t len);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `erase_addr` | `uint32_t` | Flash 擦除起始地址 |
| `len` | `uint32_t` | 擦除长度（字节） |

**Returns:**

| 返回值 | 说明 |
|--------|------|
| `0` | 擦除成功 |
| `-1` | IPC 同步超时或失败 |

**说明:** 操作流程：
1. 发送 `IPC_SYNC_SUSPEND_CMD` 并等待 NP 核心 `IPC_SYNC_SUSPEND_ACK`（超时 3 秒）
2. 调用 `bflb_flash_erase()` 执行实际擦除
3. 发送 `IPC_SYNC_RESUME_CMD` 并等待 NP 核心 `IPC_SYNC_RESUME_ACK`（超时 3 秒）
4. 若步骤 2 失败，仍会发送 Resume 恢复 NP（超时 1 秒）

---

### bflb_flash_write_mcs

多核安全的 Flash 写入操作。

```c
int bflb_flash_write_mcs(uint32_t write_addr, const uint8_t *data, uint32_t len);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `write_addr` | `uint32_t` | Flash 写入起始地址 |
| `data` | `const uint8_t *` | 待写入数据缓冲区指针 |
| `len` | `uint32_t` | 写入长度（字节） |

**Returns:**

| 返回值 | 说明 |
|--------|------|
| `0` | 写入成功 |
| `-1` | IPC 同步超时或失败 |

**说明:** 操作流程与 `bflb_flash_erase_mcs` 相同，通过 IPC 同步保护后调用 `bflb_flash_write()`。

---

### bflb_flash_read_mcs

多核安全的 Flash 读取操作。

```c
int bflb_flash_read_mcs(uint32_t addr, uint8_t *data, uint32_t len);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `addr` | `uint32_t` | Flash 读取起始地址 |
| `data` | `uint8_t *` | 读取数据缓冲区指针 |
| `len` | `uint32_t` | 读取长度（字节） |

**Returns:**

| 返回值 | 说明 |
|--------|------|
| `0` | 读取成功 |
| `-1` | IPC 同步超时或失败 |

**说明:** 操作流程与写入相同，通过 IPC 同步保护后调用 `bflb_flash_read()`。

---

### bflb_sys_reboot_mcs

多核安全的系统复位。

```c
void bflb_sys_reboot_mcs(void);
```

**说明:** 操作流程：
1. 发送 `IPC_SYNC_SUSPEND_CMD` 挂起 NP 核心（超时 3 秒）
2. 调用 `bl_sys_reset_por()` 执行系统上电复位
3. 发送 `IPC_SYNC_RESUME_CMD` 尝试恢复 NP 核心

> **注意:** 系统复位后 AP 核心将重新启动，Resume 操作实际上仅在复位未立即生效时执行。

---

## IPC 同步常量

```c
#define IPC_SYNC_SUSPEND_CMD   // NP 挂起命令
#define IPC_SYNC_SUSPEND_ACK   // NP 挂起应答
#define IPC_SYNC_RESUME_CMD    // NP 恢复命令
#define IPC_SYNC_RESUME_ACK    // NP 恢复应答
```

**超时设置:**

| 操作 | IPC 超时 |
|------|---------|
| Suspend 等待 | 3000 ms |
| Resume 等待（正常流程） | 3000 ms |
| Resume 等待（错误恢复） | 1000 ms |

---

## Usage Examples

### Example 1: OTA 固件更新（多核安全）

```c
#include "bflb_multi_core_sync.h"
#include "bflb_flash.h"

int ota_firmware_update_mcs(uint32_t partition_addr, const uint8_t *fw_data, uint32_t fw_size)
{
    int ret;
    uint32_t erase_len = ALIGN_UP(fw_size, 4096); // 4K 对齐
    
    // 1. 擦除 Flash 分区（多核安全）
    ret = bflb_flash_erase_mcs(partition_addr, erase_len);
    if (ret != 0) {
        printf("[OTA] Erase failed: %d\n", ret);
        return ret;
    }
    
    // 2. 写入新固件（多核安全）
    ret = bflb_flash_write_mcs(partition_addr, fw_data, fw_size);
    if (ret != 0) {
        printf("[OTA] Write failed: %d\n", ret);
        return ret;
    }
    
    // 3. 验证写入
    uint8_t verify_buf[256];
    ret = bflb_flash_read_mcs(partition_addr, verify_buf, 256);
    if (ret != 0) {
        printf("[OTA] Verify read failed: %d\n", ret);
        return ret;
    }
    
    if (memcmp(verify_buf, fw_data, 256) != 0) {
        printf("[OTA] Verify mismatch!\n");
        return -1;
    }
    
    printf("[OTA] Update successful\n");
    return 0;
}
```

### Example 2: 安全系统复位

```c
#include "bflb_multi_core_sync.h"

void safe_system_reboot(void)
{
    printf("System rebooting with NP sync...\n");
    
    // 确保 NP 核心安全挂起后复位
    bflb_sys_reboot_mcs();
    
    // 不会到达这里
}
```

### Example 3: Flash 配置存储

```c
#include "bflb_multi_core_sync.h"

#define CONFIG_FLASH_ADDR  0x1F0000
#define CONFIG_SECTOR_SIZE 4096

typedef struct {
    uint32_t magic;
    uint32_t version;
    uint8_t  settings[256];
    uint32_t crc;
} device_config_t;

int save_config_mcs(const device_config_t *config)
{
    int ret;
    
    // 1. 擦除配置扇区
    ret = bflb_flash_erase_mcs(CONFIG_FLASH_ADDR, CONFIG_SECTOR_SIZE);
    if (ret != 0) return ret;
    
    // 2. 写入配置
    ret = bflb_flash_write_mcs(CONFIG_FLASH_ADDR, 
                                (const uint8_t *)config, 
                                sizeof(device_config_t));
    return ret;
}

int load_config_mcs(device_config_t *config)
{
    return bflb_flash_read_mcs(CONFIG_FLASH_ADDR,
                               (uint8_t *)config,
                               sizeof(device_config_t));
}
```

---

## 注意事项

1. **编译条件:** 所有函数在 `#ifdef CONFIG_IPC` 条件下编译，确保在无 IPC 的平台上不会引入未定义引用。

2. **超时处理:** Suspend/Resume 操作有超时保护（3 秒），超时后会打印错误信息并返回 `-1`。应用层应检查返回值并妥善处理超时情况。

3. **错误恢复:** Flash 操作失败时，函数仍会尝试发送 Resume 命令恢复 NP 核心，避免 NP 永久挂起。

4. **与直接 Flash API 的区别:** `bflb_flash_erase_mcs()` / `bflb_flash_write_mcs()` / `bflb_flash_read_mcs()` 在内部调用对应的 `bflb_flash_erase()` / `bflb_flash_write()` / `bflb_flash_read()` 函数，区别在于增加了 NP 核心的 Suspend/Resume 同步保护。

5. **BL616 单核:** BL616 不需要此模块，直接使用 `bflb_flash_erase()` / `bflb_flash_write()` / `bflb_flash_read()` 即可。

| 场景 | AP 核 (BL618DG) | NP 核 (BL618DG) | BL616 |
|------|----------------|-----------------|-------|
| Flash 擦除 | `bflb_flash_erase_mcs()` | — | `bflb_flash_erase()` |
| Flash 写入 | `bflb_flash_write_mcs()` | — | `bflb_flash_write()` |
| Flash 读取 | `bflb_flash_read_mcs()` | — | `bflb_flash_read()` |
| 系统复位 | `bflb_sys_reboot_mcs()` | — | `bl_sys_reset_por()` |
