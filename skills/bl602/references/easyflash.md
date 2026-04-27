# EasyFlash KV 存储 API 参考

> 来源文件：`components/stage/easyflash4/inc/easyflash.h`  
> 基于 Flash 的键值对存储系统，支持 ENV 环境变量、日志和 IAP 升级。

---

## 概述

EasyFlash 是一个 Flash 存储管理库，提供三种功能：
- **ENV**：键值对存储（类似 NVS），掉电不丢失
- **LOG**：循环日志存储
- **IAP**：应用原地升级

---

## 头文件

```c
#include "easyflash.h"
```

---

## 类型定义

### `EfErrCode`

错误码类型：

```c
typedef enum {
    EF_NO_ERR = 0,
    EF_ERROR = 1,
    EF_NO_INIT = 2,
    EF_READ_ERR = 3,
    EF_WRITE_ERR = 4,
    /* ... 更多错误码 */
} EfErrCode;
```

### `env_node_obj_t`

ENV 节点对象句柄：

```c
typedef struct _env_node_obj {
    struct _env_node_obj *next;
    char *key;
    uint8_t len;
    uint8_t state;
    char value[ENV_NODE_VALUE_SIZE];
} env_node_obj_t;
```

---

## 初始化

### `easyflash_init`

初始化 EasyFlash（自动加载 ENV）。

```c
EfErrCode easyflash_init(void);
```

**返回值**：0=成功

---

## ENV 存储（键值对）

ENV 功能需开启 `EF_USING_ENV`。

### `ef_get_env_blob`

读取 ENV 值（二进制安全）：

```c
size_t ef_get_env_blob(const char *key, void *value_buf,
                       size_t buf_len, size_t *saved_value_len);
```

| 参数 | 说明 |
|------|------|
| `key` | 键名 |
| `value_buf` | 接收缓冲区 |
| `buf_len` | 缓冲区大小 |
| `saved_value_len` | 实际存储的长度（输出） |

**返回值**：0=键不存在，>0=读取的字节数

---

### `ef_set_env_blob`

写入 ENV 值（二进制安全）：

```c
EfErrCode ef_set_env_blob(const char *key, const void *value_buf, size_t buf_len);
```

---

### `ef_get_env`

读取 ENV 字符串值：

```c
char *ef_get_env(const char *key);
```

**返回值**：字符串指针，不存在返回 NULL

---

### `ef_set_env`

设置 ENV 字符串值：

```c
EfErrCode ef_set_env(const char *key, const char *value);
```

---

### `ef_del_env`

删除 ENV：

```c
EfErrCode ef_del_env(const char *key);
```

---

### `ef_save_env`

将内存中的 ENV 写入 Flash：

```c
EfErrCode ef_save_env(void);
```

> `ef_set_env` 默认只写内存，需手动调用 `ef_save_env` 持久化。

---

### `ef_set_and_save_env`

设置并立即保存：

```c
EfErrCode ef_set_and_save_env(const char *key, const char *value);
```

---

### `ef_print_env`

打印所有 ENV（调试用）：

```c
void ef_print_env(void);
```

---

### `ef_env_set_default`

恢复默认 ENV：

```c
EfErrCode ef_env_set_default(void);
```

---

## 日志功能

LOG 功能需开启 `EF_USING_LOG`。

### `ef_log_write`

写入日志：

```c
EfErrCode ef_log_write(const uint32_t *log, size_t size);
```

---

### `ef_log_read`

读取日志：

```c
EfErrCode ef_log_read(size_t index, uint32_t *log, size_t size);
```

---

### `ef_log_clean`

清除所有日志：

```c
EfErrCode ef_log_clean(void);
```

---

### `ef_log_get_used_size`

获取已用日志空间：

```c
size_t ef_log_get_used_size(void);
```

---

## 工具函数

### `ef_calc_crc32`

计算 CRC32：

```c
uint32_t ef_calc_crc32(uint32_t crc, const void *buf, size_t size);
```

---

## 使用示例

### 基本 ENV 操作

```c
#include "easyflash.h"

void env_demo(void)
{
    easyflash_init();

    // 写入字符串
    ef_set_env("device_name", "WB2-001");
    ef_set_env("interval", "5000");
    ef_save_env();

    // 读取字符串
    char *name = ef_get_env("device_name");
    if (name) {
        printf("Device: %s\r\n", name);
    }

    // 写入二进制数据
    uint8_t config[4] = {0x01, 0x02, 0x03, 0x04};
    ef_set_env_blob("config", config, sizeof(config));
}
```

### ENV 批量操作

```c
// 带回调的打印（遍历所有 ENV）
void my_print_cb(env_node_obj_t *env, void *arg1, void *arg2)
{
    (void)arg1; (void)arg2;
    printf("key=%s, value=%s\r\n", env->key, env->value);
}

ef_print_env_cb(my_print_cb);
```
