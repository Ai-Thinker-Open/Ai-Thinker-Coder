# CLI 命令行接口 API 参考

> 来源文件：`components/stage/cli/cli/include/cli.h`  
> 轻量级命令行接口，支持静态/动态命令注册、历史记录、自动补全（部分配置）。

---

## 概述

CLI 是 BL602 的命令行调试工具，通过 `aos_cli_register_command` 注册命令后，可在串口终端输入命令执行对应函数。支持静态命令（编译时确定）和动态命令（运行时添加/移除）。

```
> help
> mycommand arg1 arg2
Command executed with result: 0
```

---

## 头文件

```c
#include "cli.h"
```

---

## 类型定义

### `cli_command`

注册命令的结构体，每个命令包含名称、帮助文本和函数指针：

```c
struct cli_command {
    const char *name;      // 命令名，如 "reboot"
    const char *help;      // 帮助文本，如 "Reboot the device"

    void (*function)(char *pcWriteBuffer, int xWriteBufferLen,
                     int argc, char **argv);
};
```

---

## 函数接口

### `aos_cli_init`

初始化 CLI（创建任务）。

```c
int aos_cli_init(int use_thread);
```

| 参数 | 说明 |
|------|------|
| `use_thread` | 1=创建独立任务，0=同步模式 |

**返回值**：0=成功

---

### `aos_cli_register_command`

注册单个命令。

```c
int aos_cli_register_command(const struct cli_command *command);
```

**返回值**：0=成功

---

### `aos_cli_register_commands`

批量注册命令。

```c
int aos_cli_register_commands(const struct cli_command *commands, int num_commands);
```

| 参数 | 说明 |
|------|------|
| `commands` | 命令数组 |
| `num_commands` | 命令数量 |

**返回值**：0=成功

---

### `aos_cli_unregister_command`

注销单个命令。

```c
int aos_cli_unregister_command(const struct cli_command *command);
```

**返回值**：0=成功

---

### `aos_cli_unregister_commands`

批量注销命令。

```c
int aos_cli_unregister_commands(const struct cli_command *commands, int num_commands);
```

---

### `aos_cli_stop`

停止 CLI 任务并清理。

```c
int aos_cli_stop(void);
```

---

### `aos_cli_printf`

CLI 专用打印（输出到命令行缓冲区）。

```c
int aos_cli_printf(const char *buff, ...);
// 或通过宏 cmd_printf
cmd_printf("Result: %d\r\n", value);
```

---

### `aos_cli_task_create`

创建 CLI 任务（可独立调用）。

```c
int aos_cli_task_create(void);
```

---

### `aos_cli_task_get`

获取 CLI 任务句柄。

```c
void *aos_cli_task_get(void);
```

---

### `aos_cli_input_direct`

直接向 CLI 输入数据（旁路串口，用于自动化测试）。

```c
void aos_cli_input_direct(char *buffer, int count);
```

---

## 宏说明

### `cmd_printf`

CLI 内专用打印宏（自动写入输出缓冲区）：

```c
cmd_printf("Value=%d\r\n", value);
```

> 与普通 `printf` 不同，`cmd_printf` 将数据写入 CLI 输出缓冲区，适合在命令回调函数中使用。

---

## 使用示例

### 定义并注册命令

```c
#include "cli.h"

static void my_cmd_handler(char *pcWriteBuffer, int xWriteBufferLen,
                           int argc, char **argv)
{
    if (argc < 2) {
        cmd_printf("Usage: mycmd <arg1>\r\n");
        return;
    }
    cmd_printf("arg1 = %s\r\n", argv[1]);
}

// 定义静态命令
static const struct cli_command my_commands[] = {
    {
        .name = "mycmd",
        .help = "My custom command",
        .function = my_cmd_handler,
    },
};

// 注册命令
aos_cli_register_commands(my_commands, 1);
```

### 带参数解析的命令

```c
static void set_led_handler(char *pcWriteBuffer, int xWriteBufferLen,
                            int argc, char **argv)
{
    if (argc != 3) {
        cmd_printf("Usage: led <on|off> <color>\r\n");
        return;
    }

    const char *action = argv[1];  // "on" or "off"
    const char *color = argv[2];   // "red", "green", "blue"

    if (strcmp(action, "on") == 0) {
        cmd_printf("LED %s ON\r\n", color);
    } else {
        cmd_printf("LED %s OFF\r\n", color);
    }
}

static const struct cli_command led_commands[] = {
    {
        .name = "led",
        .help = "Control LED: led <on|off> <color>",
        .function = set_led_handler,
    },
};

aos_cli_register_command(&led_commands[0]);
```

### 静态命令属性

定义在特定段中的命令可被自动注册（无需手动调用注册函数）：

```c
// 编译时自动放入 .static_cli_cmds 段
static const struct cli_command hello_cmd
    __attribute__((used, section(".static_cli_cmds"))) = {
    .name = "hello",
    .help = "Say hello",
    .function = hello_handler,
};
```
