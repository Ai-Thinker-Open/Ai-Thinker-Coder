# Blog 日志系统 API 参考

> 来源文件：`components/stage/blog/blog.h`  
> BL602 日志框架，支持组件级/文件级/私有日志分区控制，带十六进制 dump 和彩色输出。

---

## 概述

Blog 是 BL602 的日志系统，核心是一组 C 宏，编译时通过 `__attribute__((section))` 将日志级别信息放到独立段，支持组件级和文件级动态日志开关。

```
blog_info("Hello %s", "world");
// 输出: INFO (100)[main.c:  42] Hello world
```

---

## 头文件

```c
#include "blog.h"
```

---

## 日志级别宏

### 组件级日志（按组件名开关）

| 宏 | 说明 | 示例输出 |
|----|------|---------|
| `blog_debug("msg")` | 调试级别 | `DEBUG (tick)[file:  42] msg` |
| `blog_info("msg")` | 信息级别 | `INFO (tick)[file:  42] msg` |
| `blog_warn("msg")` | 警告级别 | `WARN (tick)[file:  42] msg` |
| `blog_error("msg")` | 错误级别 | `ERROR (tick)[file:  42] msg` |
| `blog_assert(expr)` | 断言（失败死机） | `assert, file:42` |

### Raw 格式（无彩色前缀）

| 宏 | 说明 |
|----|------|
| `blog_debug_raw("msg")` | Raw 调试 |
| `blog_info_raw("msg")` | Raw 信息 |
| `blog_warn_raw("msg")` | Raw 警告 |
| `blog_error_raw("msg")` | Raw 错误 |

### 私有分区日志（可按模块名精细控制）

| 宏 | 说明 |
|----|------|
| `blog_debug_user(name, "msg")` | 私有调试 |
| `blog_info_user(name, "msg")` | 私有信息 |
| `blog_warn_user(name, "msg")` | 私有警告 |
| `blog_error_user(name, "msg")` | 私有错误 |

Raw 版本：`blog_debug_user_raw` / `blog_info_user_raw` / `blog_warn_user_raw` / `blog_error_user_raw`

### Hex Dump

| 宏 | 说明 |
|----|------|
| `blog_debug_hexdump(name, buf, size)` | Hex dump 调试 |
| `blog_info_hexdump(name, buf, size)` | Hex dump 信息 |
| `blog_warn_hexdump(name, buf, size)` | Hex dump 警告 |
| `blog_error_hexdump(name, buf, size)` | Hex dump 错误 |

---

## 私有分区声明

在使用 `blog_xxx_user` 系列宏前，需要声明私有分区：

```c
BLOG_DECLARE(my_module);
```

该宏展开为两个弱符号定义（可被覆盖）：
- `blog_level_t _fsymp_level_my_module` （日志级别）
- `const blog_info_t _fsymp_info_my_module` （模块信息）

---

## 函数接口

### `blog_init`

初始化日志系统。在系统启动时调用。

```c
void blog_init(void);
```

---

### `blog_set_level_log_component`

运行时动态设置组件/文件的日志级别。

```c
int blog_set_level_log_component(char *level, char *component_name);
```

| 参数 | 说明 |
|------|------|
| `level` | 日志级别字符串：`DEBUG` `INFO` `WARN` `ERROR` |
| `component_name` | 组件名或文件名 |

**返回值**：0=成功

---

### `blog_hexdump_out`

底层十六进制输出函数（被 Hex Dump 宏调用）。

```c
void blog_hexdump_out(const char *name, uint8_t width, uint8_t *buf, uint16_t size);
```

---

## 使用示例

### 基本日志

```c
#include "blog.h"

void my_task(void *param)
{
    (void)param;
    blog_info("Task started");
    blog_debug("Debug info: value=%d", 42);
    blog_warn("Warning: low memory");
    blog_error("Error: connection failed");
}
```

### 私有分区日志

```c
// 在文件开头声明私有分区
BLOG_DECLARE(my_driver);

// 使用私有日志
void driver_init(void)
{
    blog_info_user(my_driver, "Driver init");
    blog_error_user(my_driver, "HW error detected");
}
```

### Hex Dump

```c
uint8_t packet[] = {0x01, 0x02, 0x03, 0x04};
blog_info_hexdump("packet", packet, sizeof(packet));
// 输出类似:
// INFO (100)[main.c:  42] packet: 01 02 03 04
```

### 运行时调整日志级别

```c
// 禁用 DEBUG 输出（节省日志开销）
blog_set_level_log_component("INFO", "my_driver");
```
