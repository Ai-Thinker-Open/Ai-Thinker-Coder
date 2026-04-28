---
name: Ai-Thinker-Coder-bl616-618
description: 安信可 BL616/BL618 系列模组开发指南 - Wi-Fi 6 + BLE 5.0 模组，基于 bouffalo_sdk，支持 GPIO、UART、SPI、I2C、DMA、定时器、PWM、ADC、DAC、Flash、WiFi、BLE 等外设编程。使用 /skill Ai-Thinker-Coder 加载主入口。
category: hardware
tags:
  - 安信可
  - BL616
  - BL618
  - Wi-Fi 6
  - BLE 5.0
  - 物联网
  - Bouffalolab
  - 二次开发
version: 1.0.0
author: 安信可科技 & Hermes Agent
license: MIT-0
metadata:
  chip: [BL616, BL618]
  module_family: Ai-Tool
  protocols: [Wi-Fi 6, BLE 5.0]
---

# 安信可 BL616/BL618 系列开发指南

## 编程范式说明

> **重要**：除非用户明确要求采用**寄存器级（bare metal）编程**，否则本 skill 提供的代码示例均使用 **LHAL (Low Level Hardware Abstraction Layer) 库**进行开发，循环逻辑通过 **FreeRTOS 任务**实现。

**默认编程方式**（LHAL + FreeRTOS）：
- 外设初始化、配置：调用 LHAL API（如 `bflb_gpio_init`, `bflb_uart_init`）
- 业务逻辑循环：放在 FreeRTOS 任务中（`xTaskCreate`）
- 延时：`vTaskDelay`（非 `HAL_Delay`）

**BL616/BL618 FreeRTOS 特别说明**：
- **不需要调用 `vTaskStartScheduler()`**，系统会在启动代码中自动启动调度器
- `app_main()` 执行时调度器已在运行
- **主循环如果没有任何业务逻辑，必须调用 `vTaskDelay` 让出 CPU**，否则任务无法切换，系统表现为死机
- 正确的做法：`while(1) { 处理业务; vTaskDelay(pdMS_TO_TICKS(100)); }`

**寄存器级编程**（当用户明确要求时）：
- 直接操作 `*(volatile uint32_t *)addr` 访问外设寄存器
- BL616/BL618 外设基地址与 BL602 不同，需参考各章节的寄存器映射表

---

## 产品概述

**BL616/BL618** 是安信可科技基于 Bouffalolab BL616/BL618 芯片开发的 Wi-Fi 6 & BLE 5.0 双模模组，支持：

- Wi-Fi 802.11a/b/g/n/ac/ax (Wi-Fi 6)，支持 2.4GHz 和 5GHz
- Bluetooth Low Energy 5.0 + Bluetooth Mesh
- 32-bit RISC-V CPU (最高 320MHz)
- 丰富的外设接口：GPIO、UART、SPI、I2C、PWM、ADC、DAC、DMA、Timer 等
- 多种低功耗模式

### 选型表

| 型号 | 封装 | Flash | RAM | 天线 | 特点 |
|-----|------|-------|-----|------|-----|
| Ai-Tool-01F | SMD | 2MB | - | 板载 PCB | Wi-Fi 6 + BLE 5.0 |
| Ai-Tool-01M | SMD | 2MB | - | 邮票孔 | - |
| Ai-Tool-01S | SMD | 2MB | - | IPEX | - |
| Ai-Tool-12F | SMD | 4MB | - | 板载 PCB | 高性价比 |

---

## 开发环境配置（必读）

> **重要提醒**：在编写任何代码之前，必须先完成环境配置。**工具链安装和 SDK 克隆是编程的前置条件**，否则无法编译和烧录固件。

### 第一步：判断当前平台

```bash
# Windows MSYS2/MINGW 环境
echo $MSYSTEM
# 输出 MINGW64 或 MSYS 表示在 MSYS2 环境中

# Linux / WSL / macOS
uname -s
# 输出 Linux 表示 Linux/WSL，Darwin 表示 macOS
```

### 第二步：安装工具链（按平台）

#### Windows 环境

**适用场景**：Windows 原生开发，使用博流官方预编译的 RISC-V 工具链。

**第一步：下载工具链**

> ⚠️ **不要使用 MSYS2/MINGW 的 `mingw-w64-gcc`**，BL616/BL618 必须使用博流官方提供的 `riscv64-unknown-elf-gcc` 工具链。

从 GitHub 下载预编译工具链（Windows 64 位）：

- 官方链接：https://github.com/bouffalolab/toolchain_gcc_t-head_windows
- 选择 `x86_64` 目录下的最新版本（如 `x86_64-YYYY.MM.DD-mingw-w64.zip`）
- 解压到 `C:\toolchain\` 或任意目录
- **将工具链 `bin` 目录加入系统环境变量 PATH**

**验证安装**：

```powershell
# 打开「命令提示符 (CMD)」或「PowerShell」，执行：
riscv64-unknown-elf-gcc -v
# 应输出版本信息，包含 "riscv64-unknown-elf"
```

**第二步：安装 Python 和烧录工具**

```powershell
# 安装 Python3（从 python.org 或 Microsoft Store）
# 确认安装时勾选 "Add Python to PATH"

# 安装 PySerial（串口烧录用）
pip install pyserial
```

**验证安装**：

```powershell
python --version
pip show pyserial
```

> **报错则无法继续**，必须先解决工具链问题。

#### Linux (Ubuntu 20.04) 环境

**适用场景**：原生 Ubuntu、Debian、WSL2 等 Linux 环境。

**第一步：下载工具链**

博流官方 Linux 工具链：

- 官方链接：https://github.com/bouffalolab/toolchain_gcc_t-head_linux
- 选择 `x86_64-YYYY.MM.DD-linux-glibc-x86_64.tar.xz` 或类似文件
- 解压到 `/opt/toolchain/`：

```bash
cd /tmp
wget https://github.com/bouffalolab/toolchain_gcc_t-head_linux/releases/download/v1.2.0/x86_64-2024.10.08-linux-glibc-x86_64.tar.xz
sudo mkdir -p /opt/toolchain
sudo tar -xf x86_64-2024.10.08-linux-glibc-x86_64.tar.xz -C /opt/toolchain/
export PATH=/opt/toolchain/x86_64-2024.10.08-linux-glibc-x86_64/bin:$PATH

# 验证
riscv64-unknown-elf-gcc -v
```

**第二步：配置系统环境变量（永久生效）**

```bash
# 将以下内容追加到 ~/.bashrc（路径需与实际解压目录一致）
echo 'export PATH=/opt/toolchain/x86_64-2024.10.08-linux-glibc-x86_64/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

**第三步：安装依赖**

```bash
sudo apt update
sudo apt install -y build-essential git python3 python3-pip python3-dev
pip3 install pyserial
```

**验证安装**：

```bash
riscv64-unknown-elf-gcc -v
python3 --version
pip3 show pyserial
```

> **报错则无法继续**，必须先解决工具链问题。

#### macOS 环境

**第一步：下载工具链**

博流官方**不提供 macOS 预编译工具链**，需从源码编译：

```bash
# 安装依赖
brew install python3 git

# 克隆编译脚本
git clone https://github.com/p4ddy1/pine_ox64.git
cd pine_ox64
# 参考 https://github.com/p4ddy1/pine_ox64/blob/main/build_toolchain_macos.md
# 按文档编译 riscv64-unknown-elf-gcc
```

**第二步：安装 Python**

```bash
brew install python3
pip3 install pyserial
```

**验证安装**：

```bash
riscv64-unknown-elf-gcc -v
python3 --version
pip3 show pyserial
```

> macOS 工具链编译耗时较长（约 30~60 分钟），建议有条件者使用 Linux/WSL2 开发。

#### WSL2 环境

WSL2 本质上是 Linux 环境，请参考上方「Linux (Ubuntu 20.04) 环境」的安装步骤（下载 Linux 版工具链）。

> **WSL2 串口映射**：Windows 11 不支持自动串口映射，需在 Windows PowerShell（管理员）中手动映射：
> ```powershell
> # 列出设备
> usbipd list
> # 绑定并附加到 WSL
> usbipd bind --busid <busid>
> usbipd attach --wsl --busid <busid>
> ```
> 然后在 WSL 中验证 `ls /dev/ttyACM*`。

---

### 第三步：克隆 SDK（所有平台通用）

> **本地已有 SDK 的用户**：如果 SDK 已克隆到本地（如 `/path/to/bouffalo_sdk`），直接设置环境变量后跳过克隆步骤：
> ```bash
> export SDK_ROOT=/path/to/bouffalo_sdk
> ```

工具链安装完成后，所有平台执行相同的 SDK 获取步骤：

```bash
# 选择合适的目录存放项目（如 ~/projects 或 D:\workspace）
git clone https://github.com/bouffalolab/bouffalo_sdk.git
cd bouffalo_sdk

# 必须执行：检查并切换到稳定版本分支（推荐）
git checkout v1.0.0      # 或 git checkout v0.9.6 等稳定版本
# 如果不加版本号，默认 clone 的是 main 分支（可能不稳定）

# 如果需要特定芯片支持，初始化子模块
git submodule update --init --recursive
```

> **国内用户如果 GitHub 访问不稳定**，可以尝试以下方案：
> - 方案一：使用 Gitee 镜像（如果有）
> - 方案二：设置代理或使用 VPN
> - 方案三：从 https://gitee.com/bouffalolab/bouffalo_sdk 查找镜像

**Linux/WSL 额外步骤**：设置工具链权限

```bash
cd toolchain/riscv/Linux/
bash chmod755.sh
# 或手动赋权
chmod +x riscv64-elf-x86_64/bin/*
```

### 第四步：编译验证

> **Windows 用户**：必须在「MSYS2 MINGW64」终端中执行编译，**不要使用 CMD、PowerShell 或普通的 MSYS2 终端**。
> **Linux/WSL 用户**：在标准 Linux 终端中执行。

```bash
cd bouffalo_sdk

# 列出支持的芯片和开发板
ls bsp/board/     # 查看支持的开发板
ls examples/      # 查看示例项目

# 编译 Hello World 示例（BL616）
cd examples/helloworld
make CHIP=bl616 BOARD=bl616dk -j8

# 编译 Hello World 示例（BL618）
make CHIP=bl618 BOARD=bl618dk -j8
```

编译成功会在 `build/out/` 目录生成固件文件（`.bin`）。

### 第五步：烧录验证

#### 列出可用串口

烧录前先**主动列出**当前可用的串口，让用户确认模组连接到哪个端口：

```bash
# Linux - 列出所有串口设备
ls -l /dev/ttyUSB* /dev/ttyACM* /dev/ttyS* 2>/dev/null

# macOS - 列出 USB 串口
ls /dev/tty.usbserial* /dev/tty.usbmodem* 2>/dev/null

# Windows - 列出 COM 端口
mode | grep -i "com"
```

> **AI 应主动执行以上命令获取可用端口列表**，然后询问用户：「当前检测到以下串口，请确认模组连接的是哪一个？」

#### 烧录命令

```bash
# BL616 烧录（将 /dev/ttyUSB0 替换为实际串口名）
make flash CHIP=bl616 BOARD=bl616dk COMX=/dev/ttyUSB0 BAUDRATE=2000000

# BL618 烧录
make flash CHIP=bl618 BOARD=bl618dk COMX=/dev/ttyUSB0 BAUDRATE=2000000
```

> 详细烧录教程和工具下载，请参考 https://dev.bouffalolab.com/

---

## BL616/BL618 GPIO 寄存器编程

**前置条件**：已完成上述「开发环境配置」第二步和第三步。

**重要**：BL616/BL618 的外设基地址与 BL602 完全不同，**不要混用 BL602 的地址**。

### BL616/BL618 内存映射（外设部分）

| 外设 | 基地址 | 说明 |
|------|--------|------|
| GLB (全局控制) | `0x20000000` | 包含 GPIO 配置 |
| GPIP (通用输入) | `0x20002000` | GPIO 输入、中断等 |
| UART0 | `0x2000A000` | 串口 0 |
| UART1 | `0x2000A100` | 串口 1 |
| SPI | `0x2000A200` | SPI 主/从控制器 |
| I2C0 | `0x2000A300` | I2C 主/从控制器 |
| PWM | `0x2000A400` | PWM 输出 |
| TIMER | `0x2000A500` | 定时器/看门狗 |
| DMA | `0x2000C000` | DMA 控制器 |

### GPIO 基础地址

- **GLB_BASE = `0x20000000`**
- GPIO 配置寄存器：`GLB_BASE + 0x??`（每个引脚有独立的配置寄存器，非 BL602 的共享寄存器方式）
- GPIO 输出值：`GLB_BASE + 0x??`
- GPIO 输出使能：`GLB_BASE + 0x??`

> **注意**：BL616/BL618 的 GPIO 寄存器布局与 BL602 不同，每个引脚有独立的配置寄存器（per-pin register），而不是每 2 个引脚共享一个寄存器。请在编写寄存器级代码前，查阅 `drivers/soc/bl616/std/include/hardware/glb_reg.h` 和 `drivers/soc/bl616/std/include/bl616_glb_gpio.h`。

### GPIO 引脚限制

BL616 引脚：`GPIO0 ~ GPIO3, GPIO10 ~ GPIO17, GPIO20 ~ GPIO22, GPIO27 ~ GPIO30`
BL618 引脚：`GPIO0 ~ GPIO34`（更多引脚）

> **配置 GPIO 前**，必须通过 GLB 寄存器将对应引脚的功能选择为 **GPIO 模式**（而非 UART/SPI/I2C 等复用功能）。这一步通过 `GLB_GPIO_FUNC_SEL` 寄存器实现。

### 项目结构

`bouffalo_sdk` 中的项目结构：

```
bouffalo_sdk/
├── examples/
│   └── helloworld/              # 示例项目
│       └── main.c               # 入口文件
├── drivers/
│   ├── lhal/                    # LHAL 外设抽象层（跨芯片通用）
│   └── soc/                     # 芯片特定驱动
│       └── bl616/               # BL616 芯片驱动
├── bsp/
│   └── board/bl616dk/           # BL616 开发板配置
└── tools/                       # 烧录工具
```

---

## 固件烧录

### 烧录工具

| 工具 | 下载 | 说明 |
|-----|------|-----|
| BouffaloLab Dev Kit | https://dev.bouffalolab.com/ | 官方烧录+调试工具 |
| BL Dev Cube | https://github.com/bouffalolab/bouffalo_sdk/releases | 命令行烧录工具 |

### 烧录步骤

1. **确认串口**：参考上方「列出可用串口」
2. **进入烧录模式**：将模组的 **BOOT 引脚拉低**，然后复位
3. **执行烧录**：

```bash
make flash CHIP=bl616 BOARD=bl616dk COMX=/dev/ttyUSB0 BAUDRATE=2000000
```

4. **验证**：烧录成功后模组会自动重启，串口应有输出

---

## AT 指令开发

### AT 固件

BL616/BL618 模组支持标准 AT 指令集，具体请参考安信可提供的 AT 固件和 AT 指令集文档。

### 常用 AT 指令

```c
// 系统
AT                  // 测试命令
AT+RST              // 重启
AT+GMR              // 查询版本

// Wi-Fi
AT+CWMODE=1         // 设置 station 模式
AT+CWLAP            // 扫描热点
AT+CWJAP="SSID","PASSWORD"  // 连接 Wi-Fi
AT+CWQAP            // 断开 Wi-Fi
AT+CIFSR            // 查询 IP 地址

// TCP/UDP
AT+CIPSTART="TCP","192.168.1.100",8080  // 建立 TCP 连接
AT+CIPSEND=10       // 发送数据
AT+CIPCLOSE         // 关闭连接

// BLE
AT+BLEINIT=1        // 初始化 BLE
AT+BLEADVISENABLE=1 // 开始广播
AT+BLESEND=5,"12345"  // 发送数据
```

---

## 二次开发 - 外设编程

> 完整 API 函数签名和类型定义请查阅 `references/` 目录下的独立文档。
> 本章通过具体案例演示外设的初始化和操作流程。

### GPIO

```c
#include "bflb_gpio.h"

struct bflb_device_s *gpio;

// 初始化 GPIO
gpio = bflb_device_get_by_name("gpio");
bflb_gpio_init(gpio, GPIO_PIN_0, GPIO_MODE_OUTPUT_PP, GPIO puxx_pull_up);
bflb_gpio_init(gpio, GPIO_PIN_1, GPIO_MODE_INPUT, GPIO_PULL_UP);

// 写 GPIO
bflb_gpio_set(gpio, GPIO_PIN_0);  // 置高
bflb_gpio_reset(gpio, GPIO_PIN_0);  // 置低

// 读 GPIO
uint32_t value = bflb_gpio_read(gpio, GPIO_PIN_1);

// 切换
bflb_gpio_toggle(gpio, GPIO_PIN_0);
```

### UART

```c
#include "bflb_uart.h"

struct bflb_device_s *uart;

// 初始化 UART
uart = bflb_device_get_by_name("uart0");
struct bflb_uart_config_s cfg = {
    .data_bits = UART_DATA_BITS_8,
    .stop_bits = UART_STOP_BITS_1,
    .parity = UART_PARITY_NONE,
    .bit_order = UART_BIT_ORDER_LSB,
    .flow_control = UART_FLOW_CONTROL_NONE,
    .baudrate = 115200,
};
bflb_uart_init(uart, &cfg);

// 发送数据
bflb_uart_putchar(uart, 'A');

// 接收数据（轮询）
uint8_t ch;
bflb_uart_getchar(uart, &ch);
```

### DMA

```c
#include "bflb_dma.h"

struct bflb_device_s *dma;
struct bflb_dma_channel_lli_pool_s lli_pool[1];
struct bflb_dma_channel_lli_transfer_s transfer;

dma = bflb_device_get_by_name("dma0");
bflb_dma_channel_init(dma, DMA_CH0, &dma_cfg);
bflb_dma_channel_lli_reinit(dma, DMA_CH0, lli_pool, 1);
bflb_dma_channel_lli_add_node(dma, DMA_CH0, &transfer);
bflb_dma_channel_start(dma, DMA_CH0);
```

### 定时器

```c
#include "bflb_timer.h"

struct bflb_device_s *timer;

timer = bflb_device_get_by_name("timer0");
struct bflb_timer_config_s cfg = {
    .prescale = 250,
    .counter_mode = TIMER_COUNTER_MODE_PERIODIC,
    .trigger_cmp = 0,
};
bflb_timer_init(timer, &cfg);
bflb_timer_start(timer, 1000);  // 1ms 周期

// 停止
bflb_timer_stop(timer);
```

### PWM

```c
#include "bflb_pwm.h"

struct bflb_device_s *pwm;

pwm = bflb_device_get_by_name("pwm0");
struct bflb_pwm_config_s cfg = {
    .freq = 1000,       // 1kHz
    .channel = PWM_CH0,
};
bflb_pwm_init(pwm, &cfg);
bflb_pwm_set_pulsewidth(pwm, PWM_CH0, 500);  // 50% 占空比
bflb_pwm_start(pwm);
```

### ADC

```c
#include "bflb_adc.h"

struct bflb_device_s *adc;

adc = bflb_device_get_by_name("adc");
struct bflb_adc_config_s cfg = {
    .res = ADC_RES6,        // 6-bit 分辨率（或其他）
    .chan = ADC_CHAN0,
};
bflb_adc_init(adc, &cfg);

uint16_t value;
bflb_adc_read_raw(adc, &value);
```

### SPI

```c
#include "bflb_spi.h"

struct bflb_device_s *spi;

spi = bflb_device_get_by_name("spi0");
struct bflb_spi_config_s cfg = {
    .freq = 1000000,   // 1MHz
    .role = SPI_ROLE_MASTER,
    .mode = SPI_MODE_0,
    .data_width = SPI_DATA_WIDTH_8BIT,
};
bflb_spi_init(spi, &cfg);

uint8_t tx_buf[4] = {0x01, 0x02, 0x03, 0x04};
uint8_t rx_buf[4];
bflb_spi_transfer(spi, tx_buf, rx_buf, 4);
```

### I2C

```c
#include "bflb_i2c.h"

struct bflb_device_s *i2c;

i2c = bflb_device_get_by_name("i2c0");
struct bflb_i2c_config_s cfg = {
    .freq = 400000,   // 400kHz
    .addr = 0x00,
};
bflb_i2c_init(i2c, &cfg);

// 发送
uint8_t data[2] = {0x30, 0x93};
bflb_i2c_send(i2c, 0x44, data, 2);

// 接收
uint8_t buf[6];
bflb_i2c_recv(i2c, 0x44, buf, 6);
```

---

## FreeRTOS 任务开发

### 任务创建示例

BL616/BL618 的 FreeRTOS 应用通常在 `app_main()` 中完成外设初始化后，创建业务任务。

```c
#include "FreeRTOS.h"
#include "task.h"

// 任务函数
static void my_task(void *param)
{
    (void)param;
    while (1) {
        // 业务逻辑
        printf("Task running\r\n");

        // 必须调用延时让出 CPU，否则系统死机
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void app_main(void)
{
    printf("System init\r\n");

    // 初始化外设
    // ...

    // 创建任务（优先级 5，栈 512 字）
    BaseType_t ret = xTaskCreate(
        my_task,               // 任务函数
        "my_task",             // 任务名称（仅供调试）
        512,                   // 栈深度（字）
        NULL,                  // 参数
        5,                     // 优先级（1-15）
        NULL                   // 任务句柄（不需要可填 NULL）
    );

    if (ret != pdPASS) {
        printf("Task create failed\r\n");
    }

    // BL616/BL618 不需要手动调用 vTaskStartScheduler()
    // 调度器已在系统初始化阶段自动启动
}
```

### 关键要点

| 要点 | 说明 |
|-----|------|
| **不调用 `vTaskStartScheduler()`** | 调度器由系统自动启动，`app_main()` 执行时调度器已在运行 |
| **主循环必须延时** | 裸机主循环或空闲任务中若没有任何延时，系统无法调度其他任务，表现为死机 |
| **使用 `vTaskDelay`** | 不可使用 `HAL_Delay`、`usleep` 等非 RTOS 延时 |
| **`pdMS_TO_TICKS`** | 毫秒转 Tick 数，如 `pdMS_TO_TICKS(500)` = 500ms |
| **栈深度单位** | FreeRTOS 栈深度以**字（4字节）**为单位，512 = 2048 字节 |

### 常用 FreeRTOS API

```c
// 延时（推荐）
vTaskDelay(pdMS_TO_TICKS(100));         // 延时 100ms

// 获取当前 tick 数
TickType_t now = xTaskGetTickCount();

// 删除任务
vTaskDelete(NULL);                     // 删除自己
// vTaskDelete(task_handle);          // 删除指定任务

// 挂起/恢复调度器（临界区）
vTaskSuspendAll();                     // 挂起调度器
// ... 临界区代码 ...
xTaskResumeAll();                      // 恢复调度器

// 消息队列
QueueHandle_t q = xQueueCreate(10, sizeof(uint32_t));
xQueueSend(q, &value, portMAX_DELAY);
xQueueReceive(q, &value, portMAX_DELAY);

// 二值信号量（用于中断同步）
SemaphoreHandle_t sem = xSemaphoreCreateBinary();
xSemaphoreGive(sem);
xSemaphoreTake(sem, portMAX_DELAY);
```

---

## 二次开发 - WiFi 编程

### Station 模式

```c
#include "wifi_mgmr.h"

wifi_mgmr_sta_connect_params_t params = {
    .ssid = "SSID",
    .password = "PASSWORD",
    .country_code = "CN",
};

int ret = wifi_mgmr_sta_connect(&params);
if (ret == 0) {
    printf("WiFi connected\r\n");
}
```

### SoftAP 模式

```c
#include "wifi_mgmr.h"

wifi_mgmr_ap_params_t ap_params = {
    .ssid = "BL616-Setup",
    .password = "12345678",
    .channel = 6,
};

int ret = wifi_mgmr_ap_start(&ap_params);
```

### MQTT

```c
#include "mqtt_client.h"

mqtt_client_config_t mqtt_config = {
    .broker_url = "mqtt://broker.emqx.io:1883",
    .client_id = "bl616-client",
};

mqtt_client_handle_t mqtt = mqtt_client_new(&mqtt_config);
mqtt_client_subscribe(mqtt, "/topic", callback);
mqtt_client_publish(mqtt, "/topic", "Hello", 5, 0);
```

---

## 二次开发 - BLE 编程

```c
#include "ble_lib.h"

ble_gap_adv_params_t adv_params = {
    .adv_type = BLE_GAP_ADV_TYPE_IND,
    .adv_interval = 160,  // 100ms
};

ble_stack_init();
ble_gap_adv_start(&adv_params, "BL616");
```

---

## API 参考

详细 API 文档独立存放于 `references/` 目录，共 **88 个文档**，覆盖外设驱动、系统组件、网络协议、无线通信、安全加密等全部功能。

### 外设驱动

| 文档 | 内容 |
|------|------|
| [GPIO](./references/gpio.md) | GPIO 初始化、per-pin 配置（0x188+n×4）、输出、输入、中断 |
| [UART](./references/uart.md) | UART 配置、发送、接收、ioctl、DMA、自动波特率 |
| [I2C](./references/i2c.md) | I2C 主机发送/接收、内存地址、DMA、从机模式 |
| [I2C Slave](./references/i2c_slave.md) | I2C 从机地址配置、TX/RX 中断回调 |
| [SPI](./references/spi.md) | SPI 初始化、全双工、FIFO 轮询、片选管理 |
| [DMA](./references/dma.md) | DMA 通道申请、8 通道、LLI 链、UART/SPI 握手 |
| [DMAC](./references/dmac.md) | DMA 控制器上层 API、多块传输、ping-pong 缓冲 |
| [Timer](./references/timer.md) | 硬件定时器、PWM 输出、看门狗模式、FreeRTOS 延时 |
| [PWM](./references/pwm.md) | PWM 通道、时钟分频、16 位周期/阈值、舵机控制 |
| [ADC](./references/adc.md) | 12 位 SAR ADC、8 通道、连续采样、DMA |
| [DAC](./references/dac.md) | AUDAC 音频 DAC、PWM/GPDAC 输出、立体声混音、DMA |
| [Flash](./references/flash.md) | SF_CTRL、XIP 直接读取、加密分区、Flash ID |
| [Watchdog](./references/watchdog.md) | 看门狗超时、复位产生、喂狗时序 |
| [RTC](./references/rtc.md) | HBN RTC、40 位计数器、BCD 时间、闹钟 |
| [Efuse](./references/efuse.md) | Efuse 编程控制、MAC 地址读取、启动模式 |
| [RNG](./references/rng.md) | TRNG 随机数、256 位输出、硬件自检 |
| [Touch](./references/touch.md) | 电容触摸按键、16 通道、扫描、频率跳变 |
| [I2S](./references/i2s.md) | I2S 音频接口、主/从模式、DMA、音频编解码器 |
| [IR](./references/ir.md) | 红外遥控、NEC/RC5 协议、TX/RX、FIFO |
| [Camera](./references/camera.md) | DVP 接口、MJPEG 编码/解码、几何变换 |
| [SDIO](./references/sdio.md) | SDH/SDIO3 接口、ADMA2、Wi-Fi/SD 卡 |
| [Display](./references/display.md) | DPI/DSI/DBI 显示接口、OSD 图层、帧缓冲 |
| [ACOMP](./references/comp.md) | 模拟比较器、16 通道、阈值选择、滞回配置 |
| [GMAC](./references/gmac.md) | 千兆以太网 MAC、TX/RX 描述符、MDIO |
| [EMAC](./references/emac.md) | 百兆以太网 EMAC、RMII 接口、缓冲区描述符 |
| [AUDAC](./references/audac.md) | 音频 DAC、采样率 8K-48K、音量控制、零交叉检测 |
| [bak](./references/bak.md) | 备份域寄存器、VBAT 供电、RTC/GPIO 唤醒源 |
| [DBG](./references/dbg.md) | SWD/JTAG 调试接口、密码模式、芯片 ID |

### 系统与时钟

| 文档 | 内容 |
|------|------|
| [Clock](./references/clock.md) | CPU 频率、PLL 配置、时钟门控、外设时钟源 |
| [Reset](./references/reset.md) | 外设复位控制、36+ 复位编号映射表 |
| [IRQ](./references/irq.md) | ECLIC 中断控制器、注册、启用/禁用、优先级 |
| [Core](./references/core.md) | CPU 核心访问、mtime 计时器、CLINT |
| [PM](./references/pm.md) | 电源管理、深度睡眠、休眠、时钟门控、Wi-Fi 功耗 |
| [FreeRTOS](./references/freertos.md) | 任务、队列、信号量、互斥锁、滴答延时（无需 vTaskStartScheduler） |

### 安全加密

| 文档 | 内容 |
|------|------|
| [sec_sha](./references/sec_sha.md) | SHA-1/224/256/384/512 硬件加速、DMA 链接模式 |
| [sec_aes](./references/sec_aes.md) | AES ECB/CBC/CTR 模式、硬件密钥、链接模式 |
| [sec_other](./references/sec_other.md) | TRNG/DSA/GMAC/ECDSA/PKA 完整 API 参考 |
| [HMAC](./references/hmac.md) | HMAC-SHA256 软件实现（无独立硬件）、寄存器级优化 |

### 网络协议栈

| 文档 | 内容 |
|------|------|
| [LwIP](./references/lwip.md) | TCP/IP 协议栈、Socket API、UDP/TCP、netif |
| [MQTT](./references/mqtt.md) | MQTT 客户端、QoS 0/1/2、LWT、keep-alive |
| [HTTP Client](./references/http.md) | HTTP/HTTPS 客户端、GET/POST、Zephyr net/http、mbedtls 集成 |
| [HTTPD Server](./references/httpd.md) | HTTP 服务器、CGI 动态路由、SSI 标签替换、POST 处理、lwIP 内置 |
| [mbedtls](./references/mbedtls.md) | TLS 1.0-1.3、SSL 上下文、证书验证、双向认证 |
| [AT](./references/at.md) | AT 命令框架、命令注册、Wi-Fi/BLE/MQTT/HTTP AT |
| [netbus](./references/netbus.md) | 透传模式、UART-WiFi 桥接、Socket 客户端/服务端 |
| [RTSP](./references/rtsp.md) | RTSP 服务器、DESCRIBE/SETUP/PLAY、RTP over UDP |
| [NetHub](./references/nethub.md) | 推流服务器、RTSP/HTTP-FLV/HLS、Wi-Fi 包过滤分发 |
| [SRTP](./references/srtp.md) | SRTP 安全实时传输、AES-CM/GCM、ROC 同步 |
| [HTTPS](./references/https.md) | HTTPS Client、TLS 1.2/1.3、mbedtls 集成 |
| [iPerf](./references/iperf.md) | TCP/UDP 吞吐量测试、Wi-Fi 性能验证 |
| [SmartAudio](./references/smart_audio.md) | BL618 统一音频框架、本地/蓝牙音乐、提示音、音量管理 |
| [XAV](./references/xav.md) | 多媒体框架、player/codec/format/filter、MP3/AAC 播放 |
| [WebSocket](./references/websocket.md) | librws WebSocket 客户端、异步 I/O、ws://wss:// |

### 无线通信（Wi-Fi）

| 文档 | 内容 |
|------|------|
| [wifi_mgmr](./references/wifi_mgmr.md) | Wi-Fi 管理器、STA/AP 连接、扫描、国家码、自动重连 |
| [wpa_supplicant](./references/wpa_supplicant.md) | WPA2-Personal/Enterprise、WPA3、WPS-PBC/PIN、DPP、RRM/WNM |
| [net80211](./references/net80211.md) | Wi-Fi MLME 层、scan/auth/assoc、beacon 监控、rx/tx 帧处理 |
| [coex](./references/coex.md) | Wi-Fi/BLE 共存、优先级配置、活动通报 |
| [Wi-Fi 6](./references/wifi6.md) | 802.11ax、MU-MIMO、OFDMA、BSS Color、TWT |
| [Wi-Fi 4](./references/wifi4.md) | 802.11n 兼容、MCS 0~15、Short GI、APSD 节能 |
| [MACSW](./references/macsw.md) | MAC 软件层、帧收发控制、加密引擎、固件 API |
| [wl80211](./references/wl80211.md) | wl80211 驱动接口、cntrl 命令、scan_ops、input_cb |

### 无线通信（BT & Thread）

| 文档 | 内容 |
|------|------|
| [BLE](./references/ble.md) | BLE 控制器、iBeacon、GATT 服务、通知/指示 |
| [ble_mesh](./references/ble_mesh.md) | BLE Mesh 配网、模型消息发送、节点角色 |
| [bt_a2dp](./references/bt_a2dp.md) | A2DP 音频、SBC 编解码、AVRCP 遥控、Stream 管理 |
| [bt_hfp_spp](./references/bt_hfp_spp.md) | HFP 免提、AT 命令、SPP 串口、SDP 服务发现、RFCOMM |
| [openthread](./references/openthread.md) | OpenThread 协议、IPv6、CoAP、UDP Socket、设备角色 |
| [bt_avrcp](./references/bt_avrcp.md) | BT AVRCP 遥控、播放控制、歌曲信息浏览、事件通知 |
| [bt_sdp](./references/bt_sdp.md) | BT SDP 服务发现、Browse Group、Attribute 查询 |
| [lmac154](./references/lmac154.md) | 802.15.4 MAC、Thread/Zigbee 基带、频道 11~26 |
| [Matter](./references/matter.md) | Matter 智能家居、DAC 证书链、SPAKE2+ 认证、配网 |

### 组件与中间件

| 文档 | 内容 |
|------|------|
| [lvgl](./references/lvgl.md) | LVGL v9 图形库、对象系统、显示/输入驱动、定时器 |
| [Shell](./references/shell.md) | 命令行接口、SHELL_CMD_EXPORT、内置命令 |
| [Filesystem](./references/filesystem.md) | FATFS/LittleFS 文件系统、分区挂载、读写 |
| [OTA](./references/ota.md) | HTTP/HTTPS OTA 固件升级、TCP OTA、rollback |
| [USB](./references/usb.md) | USB 设备/主机、CDC ACM、MSC 类驱动 |
| [mquickjs](./references/mquickjs.md) | QuickJS JavaScript 引擎、嵌入式 JS 脚本执行 |
| [Utils](./references/utils.md) | cJSON、log 系统、日志级别、DBG_TAG |
| [AI](./references/ai.md) | DNN 加速器、图像识别、语音唤醒、手势识别 |
| [MTD](./references/mtd.md) | Flash 分区抽象、XIP 访问、PSM 持久存储 |
| [Thread](./references/thread.md) | 802.15.4 Thread 网络、IPv6 mesh、自愈合 |
| [FOTA](./references/fota.md) | 固件 OTA 升级、TCP/HTTPS、双分区回滚 |

### 多媒体与音频编解码

| 文档 | 内容 |
|------|------|
| [SmartAudio](./references/smart_audio.md) | 统一音频框架、本地/蓝牙音乐、提示音、音量管理 |
| [XAV](./references/xav.md) | 多媒体框架、player/codec/format/filter |
| [AAC](./references/aacdec.md) | AAC-LC/AAC+/eAAC+ 解码、PVMP4AudioDecoder |
| [Opus](./references/opus.md) | Opus 编解码、VoIP/音乐、SILK+CELT、6~510kbps |
| [Speex](./references/speex.md) | Speex 语音编解码、VoIP、NB/WB/UWB |
| [Vorbis](./references/vorbis.md) | Vorbis 音频解码、OGG 容器、Xiph.Org |
| [AMR](./references/amr.md) | AMR-NB/WB 语音解码、3GPP 标准、VoIP |
| [FVAD](./references/fvad.md) | 语音活动检测、WebRTC VAD、语音识别前端 |
| [AudioCodec](./references/audio_codec.md) | 声卡驱动 SndBl616、AudioFlowctrlBridge、SBC→PCM |
| [FrogFS](./references/frogfs.md) | 轻量级只读文件系统、LVGL 资源存储、XIP |

---

## 开发教程

### 入门篇

| 教程 | 链接 |
|-----|------|
| BL616 概述 | https://dev.bouffalolab.com/ |
| GPIO 使用 | 参考 `examples/peripherals/gpio` |
| UART 数据收发 | 参考 `examples/peripherals/uart` |
| DMA 数据传输 | 参考 `examples/peripherals/dma` |

### 进阶篇

| 教程 | 链接 |
|-----|------|
| Wi-Fi Station 连接 | 参考 `examples/wifi/wifi_station` |
| BLE 广播 | 参考 `examples/ble/ble_peripheral` |
| MQTT 连接 | 参考 `examples/net/mqtt` |

---

## 芯片参考手册

| 文档 | 链接 |
|-----|------|
| BL616 Datasheet | https://dev.bouffalolab.com/media/doc/bl616/datasheet |
| BL618 Datasheet | https://dev.bouffalolab.com/media/doc/bl618/datasheet |
| BL616 Reference Manual | https://dev.bouffalolab.com/media/doc/bl616/reference_manual |
| Bouffalo SDK 文档 | https://dev.bouffalolab.com/ |

---

## FAQ 常见问题

**Q: 编译报错 "No such file or directory"**
```bash
# 检查 SDK 是否克隆完整
ls bouffalo_sdk/components/
# 如果 components 目录为空或很少，重新 clone
git submodule update --init --recursive
```

**Q: 烧录失败**
- 检查 BOOT 引脚是否拉低
- 检查串口驱动是否安装
- 检查串口选择是否正确
- 尝试降低波特率（如 115200）

**Q: WiFi 连接失败**
- 检查 SSID 和密码是否正确
- 检查路由器是否支持 2.4G 频段（BL616 Wi-Fi 6 可能需要路由器支持）
- 检查国家码设置

**Q: BLE 连接不稳定**
- 检查天线是否连接正确
- 减少障碍物和干扰源

---

## 附录：BL616/BL618 vs BL602 关键差异

| 项目 | BL602 | BL616/BL618 |
|------|-------|-------------|
| CPU 频率 | 40MHz | 最高 320MHz |
| Wi-Fi | 802.11b/g/n | 802.11a/b/g/n/ac/ax (Wi-Fi 6) |
| BLE | 5.0 | 5.0 + Mesh |
| GLB_BASE | 0x40000000 | **0x20000000** |
| UART0_BASE | 0x4000A000 | **0x2000A000** |
| GPIO 配置 | 每 2 引脚共享寄存器 | **每引脚独立寄存器** |
| 外设驱动 | HOSAL | **LHAL** |
| 工具链 | riscv64-unknown-elf-gcc | riscv64-unknown-elf-gcc |
