# L1C (L1 Cache) API Reference (BL616/BL618)

> **Source:** `bouffalo_sdk/drivers/lhal/include/bflb_l1c.h`  
> **Implementation:** `bouffalo_sdk/drivers/lhal/src/bflb_l1c.c`  
> **Chip Support:** BL602, BL702/BL702L, BL616/BL618, BL618DG (LP 核心除外)

## Overview

L1C 模块提供 L1 指令缓存 (I-Cache) 和数据缓存 (D-Cache) 的控制接口。在 BL616/BL618 系列芯片上，基于 RISC-V NMSIS 核心层实现，通过 ROM API 跳转或直接操作 CSI/NMSIS 接口来控制缓存。在 BL602/BL702 等旧平台上有不同的底层实现。

**主要特性：**
- I-Cache 启用/禁用/无效化
- D-Cache 启用/禁用/清除(Clean)/无效化(Invalidate)
- 指定地址范围的缓存操作
- 缓存命中/未命中计数统计
- 缓存写策略配置（WT/WB/WA）
- 关键函数位于 TCM 段以保证低延迟

## Cache Line 大小

```c
// BL616/BL616CL/BL618DG (非 LP 核心, D0 核心)
#define BFLB_CACHE_LINE_SIZE 32  // 或 64 (取决于 CPU 类型)

// 其他平台 (BL602/BL702 等)
#define BFLB_CACHE_LINE_SIZE 4   // 仅 word 对齐
```

> **注意:** BL616/BL618 平台上 D-Cache 相关操作（clean/invalidate range 等）内部会调用 `bflb_check_cache_addr()` 校验地址有效性，确保只操作可缓存地址区间。

---

## Base Address (性能计数器)

| 寄存器区域 | 基地址 | 说明 |
|-----------|--------|------|
| L1C 性能计数器 | `0x40009000` | 仅 BL702/BL702L 及部分平台 |

> BL616/BL618 通过 NMSIS core 接口操作缓存，不直接操作寄存器。

---

## API Functions

### 1. I-Cache 控制

#### bflb_l1c_icache_enable

启用指令缓存。

```c
void bflb_l1c_icache_enable(void);
```

**说明:** 内部优先调用 ROM API (`romapi_bflb_l1c_icache_enable`)，否则通过 NMSIS `EnableICache()` 使能。

---

#### bflb_l1c_icache_disable

禁用指令缓存。

```c
void bflb_l1c_icache_disable(void);
```

---

#### bflb_l1c_icache_invalid_all

无效化整个指令缓存。

```c
void bflb_l1c_icache_invalid_all(void);
```

**说明:** 位于 TCM 段。内部优先调用 ROM API，否则通过 NMSIS `MInvalICache()` 操作。

---

#### bflb_l1c_icache_invalid_range

无效化指定地址范围的指令缓存。

```c
void bflb_l1c_icache_invalid_range(void *addr, uint32_t size);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `addr` | `void *` | 起始地址 |
| `size` | `uint32_t` | 无效化范围大小（字节） |

**说明:** 位于 TCM 段。内部调用 `bflb_check_cache_addr()` 校验地址有效性后，通过 NMSIS `MInvalICacheRange()` 执行。

---

### 2. D-Cache 控制

#### bflb_l1c_dcache_enable

启用数据缓存。

```c
void bflb_l1c_dcache_enable(void);
```

---

#### bflb_l1c_dcache_disable

禁用数据缓存。

```c
void bflb_l1c_dcache_disable(void);
```

---

#### bflb_l1c_dcache_clean_all

清除（写回）整个数据缓存。

```c
void bflb_l1c_dcache_clean_all(void);
```

**说明:** 位于 TCM 段。将脏缓存行写回内存，保留缓存数据。内部通过 NMSIS `MFlushDCache()` 执行。

---

#### bflb_l1c_dcache_invalidate_all

无效化整个数据缓存。

```c
void bflb_l1c_dcache_invalidate_all(void);
```

**说明:** 位于 TCM 段。丢弃所有缓存数据（不写回）。内部通过 NMSIS `MInvalDCache()` 执行。

---

#### bflb_l1c_dcache_clean_invalidate_all

清除并无效化整个数据缓存。

```c
void bflb_l1c_dcache_clean_invalidate_all(void);
```

**说明:** 位于 TCM 段。先将脏数据写回内存再无效化。内部通过 NMSIS `MFlushInvalDCache()` 执行。这是最安全的全部缓存刷新操作。

---

#### bflb_l1c_dcache_clean_range

清除指定地址范围的数据缓存（写回）。

```c
void bflb_l1c_dcache_clean_range(void *addr, uint32_t size);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `addr` | `void *` | 起始地址 |
| `size` | `uint32_t` | 范围大小（字节） |

**说明:** 位于 TCM 段。将指定范围内的脏缓存行写回内存。

---

#### bflb_l1c_dcache_invalidate_range

无效化指定地址范围的数据缓存。

```c
void bflb_l1c_dcache_invalidate_range(void *addr, uint32_t size);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `addr` | `void *` | 起始地址 |
| `size` | `uint32_t` | 范围大小（字节） |

**说明:** 位于 TCM 段。丢弃指定范围内的缓存数据（不写回）。使用前应确保数据已写回。

---

#### bflb_l1c_dcache_clean_invalidate_range

清除并无效化指定地址范围的数据缓存。

```c
void bflb_l1c_dcache_clean_invalidate_range(void *addr, uint32_t size);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `addr` | `void *` | 起始地址 |
| `size` | `uint32_t` | 范围大小（字节） |

**说明:** 位于 TCM 段。先将脏数据写回内存再无效化，是最安全的局部缓存刷新方式。

---

### 3. 性能计数器

#### bflb_l1c_hit_count_get

获取缓存命中计数（64 位）。

```c
void bflb_l1c_hit_count_get(uint32_t *hit_count_low, uint32_t *hit_count_high);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `hit_count_low` | `uint32_t *` | 输出：命中计数低 32 位 |
| `hit_count_high` | `uint32_t *` | 输出：命中计数高 32 位 |

**说明:** 弱函数（`__WEAK`），读取 `0x40009004` 和 `0x40009008` 寄存器。仅在 BL702/BL702L 等平台有效。

---

#### bflb_l1c_miss_count_get

获取缓存未命中计数。

```c
uint32_t bflb_l1c_miss_count_get(void);
```

**Returns:** 未命中计数值（32 位）

**说明:** 弱函数（`__WEAK`），读取 `0x4000900C` 寄存器。

---

### 4. 写策略配置

#### bflb_l1c_cache_write_set

配置缓存写策略。

```c
void bflb_l1c_cache_write_set(uint8_t wt_en, uint8_t wb_en, uint8_t wa_en);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `wt_en` | `uint8_t` | Write-Through 写穿透使能 (1=使能) |
| `wb_en` | `uint8_t` | Write-Back 写回使能 (1=使能) |
| `wa_en` | `uint8_t` | Write-Allocate 写分配使能 (1=使能) |

**说明:** 弱函数（`__WEAK`），位于 TCM 段。配置寄存器 `0x40009000`：
- Bit 4: Write-Through
- Bit 5: Write-Back  
- Bit 6: Write-Allocate

仅在 BL702/BL702L 平台有效。

---

## Usage Examples

### Example 1: DMA 传输前后缓存同步

```c
#include "bflb_l1c.h"

uint8_t dma_buffer[1024] __attribute__((aligned(32)));

void dma_transfer_example(void)
{
    // 准备 DMA 发送数据
    fill_buffer(dma_buffer, sizeof(dma_buffer));
    
    // DMA 发送前：将数据从缓存写回内存
    bflb_l1c_dcache_clean_range(dma_buffer, sizeof(dma_buffer));
    
    // 启动 DMA 发送
    dma_start_transfer(dma_buffer, sizeof(dma_buffer));
    
    // ... 等待 DMA 完成 ...
    
    // DMA 接收前：无效化缓存以避免读取到旧数据
    bflb_l1c_dcache_invalidate_range(dma_buffer, sizeof(dma_buffer));
    
    // 现在读取 dma_buffer 将直接从内存获取最新数据
}
```

### Example 2: 固件 OTA 后刷新指令缓存

```c
#include "bflb_l1c.h"

void ota_firmware_update(void)
{
    uint32_t new_fw_addr = 0x23000000;
    uint32_t new_fw_size = 0x80000;
    
    // 写入新固件到 Flash（通过 PSRAM 等缓存区域）
    flash_write(new_fw_addr, new_firmware_data, new_fw_size);
    
    // 固件区域可能在 D-Cache 中，先写回
    bflb_l1c_dcache_clean_range((void *)new_fw_addr, new_fw_size);
    
    // 无效化 I-Cache 以确保执行新固件
    bflb_l1c_icache_invalid_range((void *)new_fw_addr, new_fw_size);
    
    // 跳转到新固件
    jump_to_firmware(new_fw_addr);
}
```

### Example 3: 缓存性能统计

```c
#include "bflb_l1c.h"

void cache_perf_test(void)
{
    uint32_t hit_low = 0, hit_high = 0;
    uint32_t miss_before, miss_after;
    
    // 记录测试前计数器
    miss_before = bflb_l1c_miss_count_get();
    
    // 执行待测操作
    data_processing_benchmark();
    
    // 记录测试后计数器
    miss_after = bflb_l1c_miss_count_get();
    bflb_l1c_hit_count_get(&hit_low, &hit_high);
    
    printf("Cache misses during test: %u\n", miss_after - miss_before);
    printf("Total hit count: 0x%08x%08x\n", hit_high, hit_low);
}
```

### Example 4: 多核环境下缓存维护

```c
#include "bflb_l1c.h"

void shared_memory_update(void *shared_buf, uint32_t size)
{
    // CPU0 更新共享内存数据
    update_data(shared_buf, size);
    
    // 写回 D-Cache，使数据对 NP 核心可见
    bflb_l1c_dcache_clean_range(shared_buf, size);
    
    // 通知 NP 核心数据已更新
    notify_np_core();
}

void shared_memory_read(void *shared_buf, uint32_t size)
{
    // NP 核心更新了数据，CPU0 读取前先无效化缓存
    bflb_l1c_dcache_invalidate_range(shared_buf, size);
    
    // 现在读取到的是 NP 核心写入的最新数据
    process_data(shared_buf, size);
}
```

---

## Operation Summary

| 操作 | I-Cache | D-Cache | Cache Line |
|------|---------|---------|------------|
| **Enable** | `bflb_l1c_icache_enable()` | `bflb_l1c_dcache_enable()` | — |
| **Disable** | `bflb_l1c_icache_disable()` | `bflb_l1c_dcache_disable()` | — |
| **Clean** | — | `bflb_l1c_dcache_clean_all()` | `bflb_l1c_dcache_clean_range()` |
| **Invalidate** | `bflb_l1c_icache_invalid_all()` | `bflb_l1c_dcache_invalidate_all()` | `bflb_l1c_icache_invalid_range()` / `_dcache_invalidate_range()` |
| **Clean+Invalidate** | — | `bflb_l1c_dcache_clean_invalidate_all()` | `bflb_l1c_dcache_clean_invalidate_range()` |

### 使用场景速查

| 场景 | 推荐操作 |
|------|---------|
| DMA 发送数据之前 | `bflb_l1c_dcache_clean_range()` |
| DMA 接收数据之后 | `bflb_l1c_dcache_invalidate_range()` |
| OTA/Flash 写入固件后 | `bflb_l1c_icache_invalid_range()` |
| 多核共享内存写入后 | `bflb_l1c_dcache_clean_range()` |
| 多核共享内存读取前 | `bflb_l1c_dcache_invalidate_range()` |
| 系统复位/初始化 | `bflb_l1c_dcache_clean_invalidate_all()` + `bflb_l1c_icache_invalid_all()` |

---

## 平台差异说明

| 平台 | I-Cache | D-Cache | 性能计数器 | 写策略配置 |
|------|---------|---------|-----------|-----------|
| BL616/BL618 | NMSIS core | NMSIS core | ❌ | ❌ |
| BL618DG (AP/NP) | NMSIS core | NMSIS core | ❌ | ❌ |
| BL702/BL702L | L1C 寄存器 | L1C 寄存器 | ✅ | ✅ |
| BL602 | 空操作 | 空操作 | ❌ | ❌ |

> BL616/BL618 上性能计数器和写策略配置函数为弱函数空实现，在 BL702/BL702L 上有效。
