# BL616/BL618 FOTA 固件空中升级

FOTA（Firmware Over-The-Air，固件空中升级）是 BL616/BL618 系列芯片的重要功能，允许设备在运行过程中通过无线网络接收并更新固件，无需通过物理接口烧录。Bouffalo SDK 提供三种 OTA 机制：

- **基础 OTA**（`ota.h`）— 核心 OTA 读写与校验逻辑
- **TCP FOTA**（`tcp_fota.h`）— 通过 TCP 协议下载固件
- **HTTPS FOTA**（`https_fota.h`）— 通过 HTTPS 协议下载固件，支持 TLS 双向认证

所有 FOTA 模式均基于 **双分区（A/B）** 设计，支持升级失败后自动或手动回滚（rollback）到备份分区，确保设备始终处于可运行状态。

## 头文件

```c
#include "ota.h"         /* 核心 OTA */
#include "tcp_fota.h"    /* TCP FOTA */
#include "https_fota.h"   /* HTTPS FOTA */
```

---

## 双分区设计

BL616/BL618 采用 A/B 双分区布局：

| 分区 | 角色 | 说明 |
|------|------|------|
| Partition A | 运行时分区 | 设备当前启动的固件所在分区 |
| Partition B | 备份分区 | 接收新固件、存放旧固件副本 |

升级流程如下：

```
[当前运行分区 A]  -->  [连接 OTA 服务器]  -->  [下载固件到分区 B]
        |                                                |
        |                                                v
        |                                    [校验 SHA256 + OTA Header]
        |                                                |
        v                                                v
[重启，系统从分区 B 启动]  <--  [更新分区表，切换启动标识到 B]
        |
 (若分区 B 启动失败)
        v
[回滚到分区 A]
```

分区表（partition table）记录每个分区的地址、大小、类型和年龄（age）等信息，由 bootloader 在启动时读取以决定从哪个分区启动。`ota_handle_t` 通过 `pt_table_stuff_config` 结构体管理分区表上下文。

---

## OTA Header 结构

所有 OTA 固件镜像必须包含 512 字节的 OTA Header：

```c
typedef struct ota_header {
    union {
        struct {
            uint8_t  header[16];         /* 固定魔数 */
            uint8_t  type[4];            /* 压缩类型："RAW" 或 "XZ" */
            uint32_t len;                /* body 长度 */
            uint8_t  pad0[8];
            uint8_t  ver_hardware[16]; /* 硬件版本 */
            uint8_t  ver_software[16];  /* 软件版本 */
            uint8_t  sha256[32];        /* 固件 SHA256 校验值 */
        } s;
        uint8_t _pad[512];
    } u;
} ota_header_t;
```

---

## 基础 OTA 接口

基础 OTA（`ota.h`）提供底层分区操作和固件写入接口，供 TCP/HTTPS FOTA 内部调用。

### `ota_start()`

```c
ota_handle_t ota_start(void);
```

初始化 OTA 会话，获取当前活跃分区信息，准备 OTA 目标分区（写入备份分区）。返回 `ota_handle_t` 句柄。

### `ota_update()`

```c
int ota_update(ota_handle_t handle, uint8_t *buf, uint32_t buf_len);
```

将固件数据写入备份分区，内部处理 Flash 擦写操作。返回 0 表示成功，负值表示错误。

### `ota_verify_hash()`

```c
int ota_verify_hash(ota_handle_t handle);
```

对已写入备份分区的固件计算 SHA256，与 OTA Header 中存储的预期值比较，验证完整性。

### `ota_finish()`

```c
int ota_finish(ota_handle_t handle, uint8_t check_hash, uint8_t reboot);
```

完成 OTA 过程：更新分区表将备份分区标记为新活跃分区，`reboot` 非零时自动重启。

### `ota_abort()`

```c
int ota_abort(ota_handle_t handle);
```

中止当前 OTA 过程，放弃已写入的固件数据，恢复到升级前状态。

### `ota_rollback()`

```c
int ota_rollback(void);
```

手动回滚到备份分区中的旧固件。向 bootloader 写入特定标记，下次重启时系统将从备份分区启动。调用后**必须手动重启设备**才能完成回滚切换。

---

## TCP FOTA

TCP FOTA 通过 TCP 连接从 OTA 服务器下载固件，适用于内网或安全性要求不高的场景。

### `tcp_fota_status_e` 状态码

```c
typedef enum {
    TCP_FOTA_SUCCESS = 0,           /* OTA 成功完成 */
    TCP_FOTA_START,                  /* FOTA 过程已启动 */
    TCP_FOTA_SERVER_CONNECT_FAIL,   /* 服务器连接失败 */
    TCP_FOTA_PROCESS_TRANSFER,       /* 正在传输固件数据 */
    TCP_FOTA_TRANSFER_FINISH,        /* 固件传输完成 */
    TCP_FOTA_IMAGE_VERIFY,           /* 正在校验固件镜像 */
    TCP_FOTA_IMAGE_VERIFY_FAIL,      /* 固件镜像校验失败 */
    TCP_FOTA_ABORT,                  /* FOTA 过程被中止 */
} tcp_fota_status_t;
```

### 配置结构

```c
struct tcp_fota_config {
    pfn_tcp_fota_t callback;  /* 状态回调函数 */
    void *user_arg;           /* 用户上下文 */
};
```

### `tcp_fota_init()`

```c
tcp_fota_handle_t tcp_fota_init(const char *ip, const char *port,
                                 const struct tcp_fota_config *config);
```

创建并初始化 TCP FOTA 会话。`ip` 不可为 NULL，`port` 传 NULL 使用默认端口 3365。

### `tcp_fota_start()`

```c
int tcp_fota_start(tcp_fota_handle_t fota);
```

在 FreeRTOS 任务中启动固件下载流程（非阻塞），内部创建独立任务执行 TCP 连接、固件接收和 OTA 写入。

### `tcp_fota_finish()` / `tcp_fota_abort()`

```c
int tcp_fota_finish(tcp_fota_handle_t fota, bool reboot);
int tcp_fota_abort(tcp_fota_handle_t fota);
```

`finish` 完成 OTA 并更新分区表，`reboot` 为 true 时自动重启；`abort` 中止正在进行的 FOTA 过程。

### 一站式接口 `tcp_fota()`

```c
int tcp_fota(const char *ip, const char *port,
             const struct tcp_fota_config *config);
```

封装了 `init` → `start`（阻塞等待）→ `finish`（自动重启）的完整流程。连接失败自动重试最多 10 次，间隔 1 秒。

### `tcp_ota_rollback()`

```c
int tcp_ota_rollback(void);
```

回滚到备份分区，调用后需手动重启设备。

### 常量

| 常量 | 值 | 说明 |
|------|----|------|
| `TCP_FOTA_DEFAULT_PORT` | "3365" | 默认端口 |
| `TCP_FOTA_MAX_RETRY` | 10 | 最大重试次数 |
| `TCP_FOTA_RETRY_DELAY_MS` | 1000 | 重试间隔（毫秒）|

---

## HTTPS FOTA

HTTPS FOTA 通过 HTTPS 协议下载固件，支持服务器端证书验证和 TLS 双向认证（mTLS），适用于高安全性要求的生产环境。

### `https_fota_status_e` 状态码

```c
typedef enum {
    HTTPS_FOTA_SUCCESS = 0,          /* OTA 成功完成 */
    HTTPS_FOTA_START,                /* FOTA 过程已启动 */
    HTTPS_FOTA_SERVER_CONNECTE_FAIL, /* 服务器连接失败 */
    HTTPS_FOTA_PROCESS_TRANSFER,      /* 正在传输固件数据 */
    HTTPS_FOTA_TRANSFER_FINISH,      /* 固件传输完成 */
    HTTPS_FOTA_IMAGE_VERIFY,          /* 正在校验固件镜像 */
    HTTPS_FOTA_IMAGE_VERIFY_FAIL,    /* 固件镜像校验失败 */
    HTTPS_FOTA_ABORT,                /* FOTA 过程被中止 */
} https_fota_status_t;
```

### 配置结构

```c
struct https_fota_config {
    pfn_https_fota_t callback;       /* 状态回调函数 */
    void *user_arg;                  /* 用户上下文 */
    const char *ca_pem;              /* 服务器 CA 证书（PEM 格式）*/
    size_t      ca_len;              /* CA 证书长度 */
    const char *client_cert_pem;    /* 客户端证书（mTLS 用）*/
    size_t      client_cert_len;    /* 客户端证书长度 */
    const char *client_key_pem;     /* 客户端私钥（mTLS 用）*/
    size_t      client_key_len;     /* 客户端私钥长度 */
};
```

- 仅验证服务器证书：设置 `ca_pem` + `ca_len`。
- mTLS 双向认证：额外设置 `client_cert_pem` / `client_key_pem` 系列字段。

### `https_fota_init()`

```c
https_fota_handle_t https_fota_init(const char *url,
                                      const struct https_fota_config *config);
```

创建并初始化 HTTPS FOTA 会话。`url` 为固件下载的 HTTPS 完整 URL（不可为 NULL），`config` 可传 NULL。

### `https_fota_start()`

```c
int https_fota_start(https_fota_handle_t fota);
```

在 FreeRTOS 任务中启动 HTTPS 固件下载流程（非阻塞）。

### `https_fota_callback_register()`

```c
int https_fota_callback_register(https_fota_handle_t fota,
                                  pfn_https_fota_t pfn, void *arg);
```

注册 FOTA 状态回调函数，可在 `init` 后、`start` 前调用。

### `https_fota_finish()` / `https_fota_abort()`

```c
int https_fota_finish(https_fota_handle_t fota, bool reboot);
int https_fota_abort(https_fota_handle_t fota);
```

`finish` 完成 OTA 并更新分区表，`reboot` 为 true 时自动重启；`abort` 中止 FOTA 过程。

### 一站式接口 `https_fota()`

```c
int https_fota(const char *url, const struct https_fota_config *config);
```

封装 `init` → `start` → `finish` 完整流程，传输完成后自动校验并更新分区表。

### `https_ota_rollback()`

```c
int https_ota_rollback(void);
```

回滚到备份分区，调用后必须手动重启设备才能完成回滚切换。

### 常量

| 常量 | 值 | 说明 |
|------|----|------|
| `HTTPS_FOTA_BUFFER_SIZE` | 4096 | 默认缓冲区大小 |
| `HTTPS_FOTA_REQUEST_TIMEOUT_MS` | 8000 | HTTP 请求默认超时（毫秒）|

---

## 代码示例：完整 TCP OTA 升级

```c
#include "tcp_fota.h"
#include "FreeRTOS.h"
#include "task.h"
#include <stdio.h>

static void fota_callback(void *arg, tcp_fota_status_t event)
{
    (void)arg;
    switch (event) {
    case TCP_FOTA_START:
        printf("[OTA] 开始升级\r\n"); break;
    case TCP_FOTA_SERVER_CONNECT_FAIL:
        printf("[OTA] 服务器连接失败\r\n"); break;
    case TCP_FOTA_PROCESS_TRANSFER:
        printf("[OTA] 正在接收固件...\r\n"); break;
    case TCP_FOTA_TRANSFER_FINISH:
        printf("[OTA] 固件接收完成，校验中...\r\n"); break;
    case TCP_FOTA_IMAGE_VERIFY:
        printf("[OTA] 镜像校验中\r\n"); break;
    case TCP_FOTA_IMAGE_VERIFY_FAIL:
        printf("[OTA] 镜像校验失败\r\n"); break;
    case TCP_FOTA_SUCCESS:
        printf("[OTA] 升级成功，即将重启\r\n"); break;
    case TCP_FOTA_ABORT:
        printf("[OTA] 升级被中止\r\n"); break;
    }
}

void start_ota_task(void)
{
    struct tcp_fota_config config = {
        .callback = fota_callback,
        .user_arg = NULL,
    };

    /* 一站式接口：连接服务器 -> 下载 -> 校验 -> 重启 */
    int ret = tcp_fota("192.168.1.100", "3365", &config);
    if (ret != 0) {
        printf("[OTA] tcp_fota 启动失败: %d\r\n", ret);
    }
}
```

HTTPS OTA 示例：

```c
#include "https_fota.h"

struct https_fota_config config = {
    .callback = https_fota_callback,
    .user_arg  = NULL,
    .ca_pem    = server_ca_cert,
    .ca_len    = strlen(server_ca_cert),
};

int ret = https_fota("https://ota.example.com/firmware/v2.0.bin", &config);
```

---

## 回滚机制

### 自动回滚

当设备从新固件（分区 B）启动失败时（Bootloader 检测到启动标记异常），自动回滚到旧固件（分区 A）。

### 手动回滚

```c
tcp_ota_rollback();   /* TCP FOTA */
https_ota_rollback();  /* HTTPS FOTA */
ota_rollback();        /* 基础 OTA */
```

> **重要**：回滚函数仅修改启动标记，调用后**必须手动重启设备**才能切换分区。建议调用后使能看门狗定时器（~5 秒），防止重启前意外断电导致状态不一致。

---

## 分区要求与限制

- **A/B 双分区设计是强制要求**：OTA 写入只能针对非活跃分区，不能在正在运行的分区上直接覆盖。
- **固件体积限制**：`HOSAL_OTA_FILE_SIZE_MAX` 默认 `0x100000`（1 MB），可自定义覆盖。
- **XZ 压缩**：`ota_header.type` 为 "XZ" 时系统自动解压后再写入 Flash。
- **分片写入**：OTA 以 4096 字节（`OTA_SLICE_SIZE`）为单位分片写入 Flash，控制内存占用。

---

## 注意事项

1. **OTA 过程中不能断电**：写入 Flash 期间断电可能导致设备变砖。
2. **回滚后必须重启**：回滚函数仅修改启动标记，分区切换发生在下一次重启。
3. **版本兼容性**：系统仅做 SHA256 完整性校验，`ver_hardware`/`ver_software` 版本兼容性判断由应用层负责。

---

## 参考

- [Bouffalo SDK 官方文档](../CLAUDE.md)
- 源码：
  - `components/fota/ota/ota.h` — 核心 OTA 接口
  - `components/fota/tcp/tcp_fota.h` — TCP FOTA 接口
  - `components/fota/https/https_fota.h` — HTTPS FOTA 接口
  - `components/fota/compat/bflb_ota.h` — 兼容层接口
