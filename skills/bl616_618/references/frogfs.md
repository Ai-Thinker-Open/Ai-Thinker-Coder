# FrogFS 嵌入式只读文件系统

## 概述

FrogFS 是一款专为嵌入式场景设计的轻量级只读文件系统，主要用于存储 LVGL 图形库所需的字体、图片等多媒体资源。该文件系统经过深度优化，完美支持 XIP（eXecute In Place）直接执行模式，数据可直接从 Flash 存储设备中读取并执行，无需全部加载到 RAM 中。

FrogFS 的设计理念是最小化资源占用，同时提供高效的文件访问能力，非常适合 BL618 等低资源嵌入式芯片的多媒体资源存储场景。

## 版本与标识

| 属性 | 值 |
|------|-----|
| 版本号 | FROGFS_VER_MAJOR.MINOR = 1.0 |
| 魔数 | FROGFS_MAGIC = 0x474F5246 ('FROG') |

文件系统镜像头部包含魔数标识，用于快速识别和验证 FrogFS 格式的有效性。

## 核心数据类型

### 条目类型

```c
typedef enum frogfs_entry_type_t {
    FROGFS_ENTRY_TYPE_DIR,    // 目录类型
    FROGFS_ENTRY_TYPE_FILE,   // 文件类型
} frogfs_entry_type_t;
```

### 文件系统句柄

```c
typedef struct frogfs_fs_t frogfs_fs_t;
```

### 文件/目录句柄

```c
typedef struct frogfs_fh_t frogfs_fh_t;  // 文件句柄
typedef struct frogfs_dh_t frogfs_dh_t;  // 目录句柄
```

### 文件信息结构

```c
typedef struct frogfs_stat_t {
    frogfs_entry_type_t type;      // 条目类型
    size_t size;                    // 解压后文件大小
    frogfs_comp_algo_t compression; // 压缩算法类型
    size_t compressed_sz;           // 压缩后大小
} frogfs_stat_t;
```

### 配置结构

```c
typedef struct frogfs_config_t {
    const void *addr;  // 文件系统在内存中的地址
} frogfs_config_t;
```

## 核心 API

### 初始化与销毁

#### frogfs_mount / frogfs_init

```c
frogfs_fs_t *frogfs_init(const frogfs_config_t *conf);
```

挂载 FrogFS 镜像，传入文件系统配置（主要是镜像在内存中的地址），返回文件系统句柄。

#### frogfs_unmount / frogfs_deinit

```c
void frogfs_deinit(frogfs_fs_t *fs);
```

卸载文件系统，释放相关资源。

### 文件操作

#### frogfs_open

```c
frogfs_fh_t *frogfs_open(const frogfs_fs_t *fs, const frogfs_entry_t *entry,
        unsigned int flags);
```

根据路径打开文件或目录。`flags` 支持 `FROGFS_OPEN_RAW` 标志，以原始方式打开文件（不解压），适用于需要传输压缩数据的场景（如通过 HTTP 传输）。

#### frogfs_close

```c
void frogfs_close(frogfs_fh_t *fh);
```

关闭已打开的文件句柄。

#### frogfs_read

```c
ssize_t frogfs_read(frogfs_fh_t *fh, void *buf, size_t len);
```

从打开的文件中读取数据，返回实际读取的字节数。

#### frogfs_lseek / frogfs_seek

```c
ssize_t frogfs_seek(frogfs_fh_t *fh, long offset, int mode);
```

移动文件指针。`mode` 可选 `SEEK_SET`（相对文件头）、`SEEK_CUR`（相对当前位置）、`SEEK_END`（相对文件尾）。

#### frogfs_tell

```c
size_t frogfs_tell(frogfs_fh_t *fh);
```

获取当前文件指针位置。

### 目录操作

#### frogfs_list / frogfs_opendir + frogfs_readdir

```c
frogfs_dh_t *frogfs_opendir(frogfs_fs_t *fs, const frogfs_entry_t *entry);
const frogfs_entry_t *frogfs_readdir(frogfs_dh_t *dh);
```

打开目录并逐个读取条目内容。

#### frogfs_closedir

```c
void frogfs_closedir(frogfs_dh_t *dh);
```

关闭目录句柄。

### 条目查询

#### frogfs_stat

```c
void frogfs_stat(const frogfs_fs_t *fs, const frogfs_entry_t *entry,
        frogfs_stat_t *st);
```

获取文件或目录的详细信息（类型、大小、压缩算法等）。

#### frogfs_get_entry

```c
const frogfs_entry_t *frogfs_get_entry(const frogfs_fs_t *fs,
        const char *path);
```

根据路径获取条目指针。

#### frogfs_is_dir / frogfs_is_file

```c
int frogfs_is_dir(const frogfs_entry_t *entry);
int frogfs_is_file(const frogfs_entry_t *entry);
```

判断条目类型。

## FROGFS_OPEN_RAW 标志

```c
#define FROGFS_OPEN_RAW (1 << 0)
```

该标志用于以原始（未解压）模式打开文件。当设置此标志时，`frogfs_read` 将返回压缩前的原始数据，适用于需要绕过文件系统解压直接获取源数据的场景，例如通过 HTTP 传输压缩文件。

## 压缩算法支持

FrogFS 支持多种压缩算法：

| 算法 ID | 说明 |
|---------|------|
| FROGFS_COMP_ALGO_NONE | 无压缩 |
| FROGFS_COMP_ALGO_ZLIB | ZLIB 压缩 |
| FROGFS_COMP_ALGO_HEATSHRINK | Heatshrink 压缩 |
| FROGFS_COMP_ALGO_GZIP | GZIP 压缩 |

## 特性

- **只读设计**：文件系统为只读，确保数据完整性，适合嵌入式存储
- **XIP 支持**：支持直接从 Flash 地址执行代码/资源，无需完整加载到 RAM
- **嵌入式优化**：最小化 RAM 占用，适合资源受限的 MCU 环境
- **压缩支持**：内置多种压缩算法，节省存储空间
- **LVGL 集成**：专为 LVGL 图形库资源文件设计

## 使用示例

### 基本挂载与读取

```c
#include "frogfs.h"

// 配置并挂载文件系统
frogfs_config_t config = {
    .addr = (void *)0x8000000,  // Flash 起始地址
};
frogfs_fs_t *fs = frogfs_init(&config);
if (fs == NULL) {
    // 挂载失败
    return -1;
}

// 打开文件
const frogfs_entry_t *entry = frogfs_get_entry(fs, "/fonts/myfont.bin");
if (entry && frogfs_is_file(entry)) {
    frogfs_fh_t *fh = frogfs_open(fs, entry, 0);
    if (fh) {
        uint8_t buffer[256];
        ssize_t len = frogfs_read(fh, buffer, sizeof(buffer));
        
        // 处理读取的数据
        // ...
        
        frogfs_close(fh);
    }
}

// 卸载文件系统
frogfs_deinit(fs);
```

### 遍历目录

```c
#include "frogfs.h"

void list_directory(frogfs_fs_t *fs, const frogfs_entry_t *dir) {
    frogfs_dh_t *dh = frogfs_opendir(fs, dir);
    if (!dh) return;
    
    const frogfs_entry_t *entry;
    while ((entry = frogfs_readdir(dh)) != NULL) {
        frogfs_stat_t st;
        frogfs_stat(fs, entry, &st);
        
        printf("Name: %s, Type: %s, Size: %zu\n",
               frogfs_get_name(entry),
               frogfs_is_dir(entry) ? "DIR" : "FILE",
               st.size);
    }
    
    frogfs_closedir(dh);
}
```

### 获取文件统计信息

```c
#include "frogfs.h"

void show_file_info(frogfs_fs_t *fs, const char *path) {
    const frogfs_entry_t *entry = frogfs_get_entry(fs, path);
    if (!entry) {
        printf("File not found: %s\n", path);
        return;
    }
    
    frogfs_stat_t st;
    frogfs_stat(fs, entry, &st);
    
    printf("Path: %s\n", path);
    printf("Type: %s\n", frogfs_is_dir(entry) ? "Directory" : "File");
    printf("Size: %zu bytes\n", st.size);
    printf("Compressed: %zu bytes\n", st.compressed_sz);
    printf("Compression: %d\n", st.compression);
}
```

## 注意事项

1. FrogFS 为只读文件系统，不支持写操作
2. 镜像文件需要预先烧录到 Flash 或其他非易失性存储中
3. 使用 XIP 功能时，需要确保 Flash 地址映射正确
4. 文件系统访问会自动处理解压（除非使用 `FROGFS_OPEN_RAW`）
5. 目录和文件名称区分大小写

## 参考

- [FrogFS 官方源码](https://github.com/lvgl/lv_lib_freetype) （LVGL 生态的一部分）
- Bouffalo SDK 组件：`components/graphics/lvgl_v9/libs/frogfs`
- 头文件：
  - `include/frogfs/frogfs.h` - 主头文件
  - `include/frogfs/frogfs_types.h` - 类型定义
