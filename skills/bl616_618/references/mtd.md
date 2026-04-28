# MTD (Memory Technology Device) 技术文档

## 概述

MTD（Memory Technology Device，内存技术设备）是 BL618/BL616 系列芯片提供的一种统一 Flash 分区抽象层。该模块位于 `bouffalo_sdk/components/utils/bflb_mtd/` 目录下，提供了对 Flash 存储器的标准化访问接口，使开发者无需关心底层 Flash 硬件的物理特性，即可完成数据的读取、写入和擦除操作。

MTD 模块的核心功能包括：

- **分区抽象**：将 Flash 划分为多个逻辑分区，每个分区拥有独立的名称、偏移地址和大小
- **XIP 地址访问**：支持直接通过 Flash 映射的 XIP（eXecute In Place）地址读取代码或数据，无需将数据拷贝到 RAM
- **备份分区支持**：支持 A/B 双分区机制，可打开备份分区进行固件升级或数据备份
- **PSM 持久存储**：提供专门的 PSM（Persistent Storage Memory）分区用于保存需要掉电保留的配置参数
- **统一错误处理**：所有 API 返回值遵循统一的错误码规范，返回 0 表示成功，负值表示错误

MTD 层介于底层 Flash 驱动和高级文件系统之间，适用于对存储空间有直接访问需求的场景，例如固件升级、参数存储、媒体数据读写等。

## 核心数据结构

### bflb_mtd_handle_t 句柄类型

```c
typedef void *bflb_mtd_handle_t;
```

`bflb_mtd_handle_t` 是 MTD 模块的 opaque（不透明）句柄类型，用于标识一个已打开的 Flash 分区。用户在调用 `bflb_mtd_open()` 成功后获得该句柄，后续所有分区操作（读、写、擦除）均需传递此句柄。句柄由 MTD 内部管理，用户不应尝试解引用或修改其值。

使用流程：

1. 调用 `bflb_mtd_init()` 初始化 MTD 子系统
2. 调用 `bflb_mtd_open()` 打开目标分区，获取句柄
3. 使用句柄进行读写操作
4. 操作完成后调用 `bflb_mtd_close()` 关闭分区

### bflb_mtd_info_t 分区信息结构体

```c
typedef struct {
    char name[16];       /*!< 分区名称 */
    unsigned int offset; /*!< 分区在 Flash 中的偏移地址 */
    unsigned int size;   /*!< 分区大小（字节） */
    void *xip_addr;      /*!< 分区的 XIP 映射地址 */
} bflb_mtd_info_t;
```

通过 `bflb_mtd_info()` 可获取指定分区的详细信息。其中 `xip_addr` 字段提供该分区在 Flash 映射区域的起始地址，可直接用于 XIP 读取场景。

## 打开标志

`bflb_mtd_open()` 函数支持以下打开标志：

| 标志名称 | 值 | 说明 |
|---------|-----|------|
| `BFLB_MTD_OPEN_FLAG_NONE` | 0 | 以默认方式打开分区 |
| `BFLB_MTD_OPEN_FLAG_BACKUP` | (1 << 0) | 打开备份分区（双分区场景） |
| `BFLB_MTD_OPEN_FLAG_BUSADDR` | (1 << 1) | 返回 Flash 总线地址而非 XIP 地址 |

在 OTA 升级等双分区场景下，可使用 `BFLB_MTD_OPEN_FLAG_BACKUP` 标志打开当前未激活的备份分区，实现新固件的下载和校验。

## 核心 API

### bflb_mtd_init

```c
void bflb_mtd_init(void);
```

初始化 MTD 子系统。在调用任何其他 MTD API 之前，必须先调用此函数。该函数会读取 Flash 分区表（Partition Table），初始化内部数据结构，准备好分区管理环境。

**示例**：

```c
bflb_mtd_init();
```

### bflb_mtd_open

```c
int bflb_mtd_open(const char *name, bflb_mtd_handle_t *handle, unsigned int flags);
```

根据分区名称打开一个 Flash 分区，获取操作句柄。

**参数说明**：

- `name`：分区名称字符串，如 `"PSM"`、`"FW"`、`"media"`
- `handle`：输出参数，成功打开后存放句柄地址
- `flags`：打开标志，详见上文"打开标志"章节

**返回值**：成功返回 0，失败返回负数错误码

**示例**：

```c
bflb_mtd_handle_t handle;
int ret = bflb_mtd_open("PSM", &handle, BFLB_MTD_OPEN_FLAG_NONE);
if (ret != 0) {
    printf("Failed to open PSM partition\r\n");
    return ret;
}
```

### bflb_mtd_close

```c
int bflb_mtd_close(bflb_mtd_handle_t handle);
```

关闭一个已打开的 Flash 分区，释放相关资源。

**参数说明**：

- `handle`：要关闭的分区句柄

**返回值**：成功返回 0，失败返回负数错误码

### bflb_mtd_info

```c
int bflb_mtd_info(bflb_mtd_handle_t handle, bflb_mtd_info_t *info);
```

获取指定分区的详细信息，包括名称、偏移地址、大小和 XIP 地址。

**参数说明**：

- `handle`：分区句柄
- `info`：输出参数，存放分区信息结构体

**返回值**：成功返回 0，失败返回负数错误码

**示例**：

```c
bflb_mtd_info_t info;
bflb_mtd_info(handle, &info);
printf("Partition: %s, Offset: 0x%x, Size: %u bytes, XIP: %p\r\n",
       info.name, info.offset, info.size, info.xip_addr);
```

### bflb_mtd_erase

```c
int bflb_mtd_erase(bflb_mtd_handle_t handle, unsigned int addr, unsigned int size);
```

擦除分区内指定地址范围的 Flash 内容。Flash 写入前必须先擦除，擦除操作以扇区为单位，实际擦除范围可能会对齐到扇区边界。

**参数说明**：

- `handle`：分区句柄
- `addr`：相对于分区起始地址的偏移（字节）
- `size`：要擦除的字节数

**返回值**：成功返回 0，失败返回负数错误码

**注意**：擦除地址和大小会自动对齐到 Flash 扇区大小。

### bflb_mtd_erase_all

```c
int bflb_mtd_erase_all(bflb_mtd_handle_t handle);
```

擦除整个分区的内容。此操作需要较长时间，应避免在中断上下文中调用。

**参数说明**：

- `handle`：分区句柄

**返回值**：成功返回 0，失败返回负数错误码

### bflb_mtd_write

```c
int bflb_mtd_write(bflb_mtd_handle_t handle, unsigned int addr, unsigned int size, const uint8_t *data);
```

向分区写入数据。写入前应确保目标区域已擦除，否则可能导致写入失败或数据错误。

**参数说明**：

- `handle`：分区句柄
- `addr`：相对于分区起始地址的写入偏移（字节）
- `size`：要写入的数据长度（字节）
- `data`：要写入的数据缓冲区指针

**返回值**：成功返回 0，失败返回负数错误码

### bflb_mtd_read

```c
int bflb_mtd_read(bflb_mtd_handle_t handle, unsigned int addr, unsigned int size, uint8_t *data);
```

从分区读取数据。该操作无需预先擦除，可随时读取任意位置的数据。

**参数说明**：

- `handle`：分区句柄
- `addr`：相对于分区起始地址的读取偏移（字节）
- `size`：要读取的数据长度（字节）
- `data`：读取数据存放的缓冲区指针

**返回值**：成功返回 0，失败返回负数错误码

### bflb_mtd_size

```c
int bflb_mtd_size(bflb_mtd_handle_t handle, unsigned int *size);
```

获取分区总大小（字节）。

**参数说明**：

- `handle`：分区句柄
- `size`：输出参数，存放分区大小

**返回值**：成功返回 0，失败返回负数错误码

## 预定义分区名

BL618/BL616 SDK 预定义以下常用分区名称：

| 分区名常量 | 字符串值 | 用途说明 |
|-----------|---------|---------|
| `BFLB_MTD_PARTITION_NAME_PSM` | `"PSM"` | 持久化存储分区，用于保存设备配置参数、校准数据等需要掉电保留的信息 |
| `BFLB_MTD_PARTITION_NAME_FW_DEFAULT` | `"FW"` | 默认固件分区，存放主应用程序代码 |
| `BFLB_MTD_PARTITION_NAME_ROMFS` | `"media"` | 媒体分区，用于存储图片、音频等媒体资源文件 |

这些分区名称与分区表（Partition Table）中的条目一一对应，分区表由 bootloader 在启动时读取。具体分区大小和地址由芯片型号和 SDK 配置决定，可在 `partition.toml` 或 `partition.h` 文件中查看和修改。

## MTD 与文件系统的区别

很多开发者会疑惑：既然有了文件系统，为什么还需要 MTD？两者的定位和适用场景有显著区别：

| 特性 | MTD | 文件系统 |
|-----|-----|---------|
| 层级 | 裸 Flash 读写接口（块设备驱动层） | 高级抽象（文件/目录/路径） |
| 典型代表 | bflb_mtd_* 系列 API | FATFS、LittleFS、ROMFS |
| 数据组织 | 线性地址空间，无结构 | 树形目录结构 |
| 随机访问 | 需要手动计算地址 | 通过文件名和偏移直接访问 |
| 适用场景 | 固件存储、PSM 参数、裸数据块 | 日志记录、配置文件、多媒体资源 |
| 实现复杂度 | 简单 | 复杂（需要管理目录项、簇链、元数据） |
| 资源开销 | 极低 | 需要额外 RAM/ROM |

简而言之：

- **MTD** 是底层接口，直接操作 Flash 物理地址，适合对存储布局有精确控制需求的场景
- **文件系统** 是高级抽象，提供类似 PC 文件系统的使用体验，适合管理大量小文件和复杂数据结构

在实际项目中，通常的架构是：Bootloader 使用 MTD 直接读写 Flash 分区表和固件镜像；应用程序使用文件系统管理用户数据；而 PSM 等关键配置区域则通过 MTD 直接访问以确保可靠性和低延迟。

## 代码示例：PSM 分区读写

以下示例演示如何打开 PSM 分区并读写持久化数据：

```c
#include "bflb_mtd.h"
#include <stdio.h>
#include <string.h>

#define PSM_CONFIG_ADDR  0
#define PSM_CONFIG_SIZE  64

typedef struct {
    uint32_t magic;
    uint32_t version;
    char device_name[32];
    uint8_t reserved[24];
} psm_config_t;

static const psm_config_t default_config = {
    .magic = 0x50534D00,  /* "PSM\0" */
    .version = 1,
    .device_name = "BL618-Dev",
};

/**
 * @brief 从 PSM 分区加载配置
 */
int psm_load_config(psm_config_t *config)
{
    bflb_mtd_handle_t handle;
    int ret;

    ret = bflb_mtd_open(BFLB_MTD_PARTITION_NAME_PSM, &handle, BFLB_MTD_OPEN_FLAG_NONE);
    if (ret != 0) {
        printf("Failed to open PSM partition: %d\r\n", ret);
        return ret;
    }

    ret = bflb_mtd_read(handle, PSM_CONFIG_ADDR, sizeof(psm_config_t), (uint8_t *)config);
    if (ret != 0) {
        printf("Failed to read PSM config: %d\r\n", ret);
        bflb_mtd_close(handle);
        return ret;
    }

    /* 检查 magic 是否匹配，若不匹配使用默认值 */
    if (config->magic != default_config.magic) {
        printf("PSM config not found, using defaults\r\n");
        memcpy(config, &default_config, sizeof(psm_config_t));
        ret = -1;  /* 表示使用了默认配置 */
    }

    bflb_mtd_close(handle);
    return ret;
}

/**
 * @brief 保存配置到 PSM 分区
 */
int psm_save_config(const psm_config_t *config)
{
    bflb_mtd_handle_t handle;
    unsigned int partition_size;
    int ret;

    ret = bflb_mtd_open(BFLB_MTD_PARTITION_NAME_PSM, &handle, BFLB_MTD_OPEN_FLAG_NONE);
    if (ret != 0) {
        printf("Failed to open PSM partition: %d\r\n", ret);
        return ret;
    }

    /* 确保写入位置在分区范围内 */
    bflb_mtd_size(handle, &partition_size);
    if (PSM_CONFIG_ADDR + sizeof(psm_config_t) > partition_size) {
        printf("PSM config out of partition range\r\n");
        bflb_mtd_close(handle);
        return -1;
    }

    /* 擦除目标区域后再写入 */
    ret = bflb_mtd_erase(handle, PSM_CONFIG_ADDR, sizeof(psm_config_t));
    if (ret != 0) {
        printf("Failed to erase PSM: %d\r\n", ret);
        bflb_mtd_close(handle);
        return ret;
    }

    ret = bflb_mtd_write(handle, PSM_CONFIG_ADDR, sizeof(psm_config_t), (const uint8_t *)config);
    if (ret != 0) {
        printf("Failed to write PSM: %d\r\n", ret);
    }

    bflb_mtd_close(handle);
    return ret;
}

/**
 * @brief 应用初始化示例
 */
void app_main(void)
{
    psm_config_t config;
    int ret;

    /* 初始化 MTD 子系统 */
    bflb_mtd_init();

    /* 加载配置 */
    ret = psm_load_config(&config);
    if (ret == 0) {
        printf("Loaded config: device_name=%s, version=%u\r\n",
               config.device_name, config.version);
    }

    /* 修改配置 */
    config.version++;
    strcpy(config.device_name, "BL618-Production");

    /* 保存配置 */
    ret = psm_save_config(&config);
    if (ret == 0) {
        printf("Config saved successfully\r\n");
    }
}
```

**代码说明**：

1. 使用 `bflb_mtd_init()` 初始化 MTD 子系统
2. 通过 `bflb_mtd_open()` 配合分区名 `BFLB_MTD_PARTITION_NAME_PSM` 打开 PSM 分区
3. 读取前不需擦除，但写入前必须调用 `bflb_mtd_erase()` 擦除目标区域
4. 操作完成后务必调用 `bflb_mtd_close()` 释放句柄
5. 示例中使用 `magic` 字段标识配置是否已写入，首次上电时自动使用默认值

## 错误码

MTD 模块使用标准错误码，返回值语义如下：

| 返回值 | 含义 |
|-------|------|
| 0 | 操作成功 |
| -1 | 通用错误 |
| -2 | 参数错误 |
| -3 | 分区不存在 |
| -4 | 读写失败 |
| -5 | 擦除失败 |
| -6 | Flash 忙或超时 |

具体错误码定义可查阅 `bflb_mtd.h` 头文件或 SDK 错误码文档。

## 线程安全性

MTD 模块本身**不是线程安全**的。在多任务（RTOS）环境下使用 MTD，应遵循以下原则：

- 避免多个任务同时操作同一个分区
- 如需在多任务间共享分区访问，应使用互斥锁（Mutex）保护
- 中断上下文不应调用耗时较长的 MTD 操作（如 `bflb_mtd_erase_all`）

## 参考

- [bflb_mtd.h 源码](../workspase/BL618Claw/bouffalo_sdk/components/utils/bflb_mtd/include/bflb_mtd.h) - MTD 模块头文件
- [bflb_boot2.h 源码](../workspase/BL618Claw/bouffalo_sdk/components/utils/bflb_mtd/include/bflb_boot2.h) - Boot2 分区管理头文件
- [BL618/BL616 SDK 文档](../workspase/BL618Claw/bouffalo_sdk/CLAUDE.md) - Bouffalo SDK 整体架构
- 分区表配置 - `partition.h` / `partition.toml`
