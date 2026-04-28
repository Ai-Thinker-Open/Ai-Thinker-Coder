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

#### Windows MSYS2 环境

**适用场景**：Windows 原生开发，使用 MSYS2 提供编译工具链。

**第一步：下载并安装 MSYS2**

1. 下载 MSYS2：https://www.msys2.org/
2. 运行安装程序，选择安装路径（建议 `C:\msys64`）
3. 安装完成后，**打开「MSYS2 MINGW64」终端**（不要用 MSYS2 原生终端）

**第二步：安装编译工具链**

在 MSYS2 MINGW64 终端中执行：

```bash
# 配置阿里云镜像源（国内加速）
echo 'Server = https://mirrors.aliyun.com/msys2/$repo' > /etc/pacman.d/mirrorlist

# 更新包数据库
pacman -Sy

# 安装 RISC-V 工具链（BL616/BL618 使用 riscv64-unknown-elf-gcc）
pacman -S mingw-w64-x86_64-gcc mingw-w64-x86_64-gdb mingw-w64-x86_64-make

# 安装 Python3（烧录工具需要）
pacman -S python3 python3-pip

# 安装 PySerial（串口烧录用）
pip install pyserial
```

> **如果阿里云镜像访问失败**，可尝试腾讯云或华为云镜像：
> ```bash
> echo 'Server = https://mirrors.cloud.tencent.com/msys2/$repo' > /etc/pacman.d/mirrorlist
> ```

**验证安装**：

```bash
riscv64-unknown-elf-gcc -v    # 应输出版本信息
python3 --version             # 应输出版本号
pip show pyserial            # 应显示 pyserial 信息
```

> **报错则无法继续**，必须先解决工具链问题。

#### Linux (Ubuntu 20.04) 环境

**适用场景**：原生 Ubuntu、Debian 等 Linux 环境。

```bash
# 更新软件源
sudo apt update

# 安装编译工具链
sudo apt install -y build-essential git python3 python3-pip python3-dev wget sed

# 安装 RISC-V 工具链（如果系统仓库没有，需从源码编译或下载预编译版本）
# 检查是否已安装
riscv64-unknown-elf-gcc -v

# 如果没有，从 GitHub 下载预编译工具链
cd /tmp
wget https://github.com/bouffalolab/toolchain/releases/download/v1.0.0/riscv64-elf-x86_64.tar.gz
sudo tar -xzf riscv64-elf-x86_64.tar.gz -C /opt/
export PATH=/opt/riscv64-elf-x86_64/bin:$PATH
```

**验证安装**：

```bash
riscv64-unknown-elf-gcc -v
python3 --version
pip3 show pyserial
```

> **报错则无法继续**，必须先解决工具链问题。

#### macOS 环境

```bash
# 使用 Homebrew 安装
brew install gcc git python3

# 安装 PySerial
pip3 install pyserial
```

**验证安装**：

```bash
gcc -v
python3 --version
pip3 show pyserial
```

---

**Windows 备选方案：已有 WSL2 的用户**

如果 Windows 上已经安装了 WSL2，可以跳过 MSYS2，直接使用 WSL2 进行开发。WSL2 本质上就是 Linux 环境，请参考上方「Linux (Ubuntu 20.04) 环境」的安装步骤。

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

详细 API 文档独立存放于 `references/` 目录，按外设分类：

### 外设驱动

| 文档 | 内容 |
|------|------|
| [GPIO](./references/gpio.md) | GPIO 初始化、输出、输入、中断（BL616/BL618 per-pin 配置） |
| [UART](./references/uart.md) | UART 配置、发送、接收、ioctl、DMA |
| [I2C](./references/i2c.md) | I2C 主机/从机发送、接收、内存读写 |
| [SPI](./references/spi.md) | SPI 初始化、发送、接收、片选 |
| [DMA](./references/dma.md) | DMA 通道申请、启停、中断回调（8 通道） |
| [Timer](./references/timer.md) | 硬件定时器初始化、启动、停止 |
| [PWM](./references/pwm.md) | PWM 输出、频率/占空比动态调节 |
| [ADC](./references/adc.md) | ADC 采样、通道管理、连续采样 |
| [DAC](./references/dac.md) | DAC 输出、电压值设置、DMA 模式 |
| [Flash](./references/flash.md) | Flash 分区读写、擦除、XIP |
| [Watchdog](./references/watchdog.md) | 看门狗初始化、喂狗 |
| [RTC](./references/rtc.md) | RTC 时间设置/读取、闹钟 |
| [Efuse](./references/efuse.md) | Efuse 一次性存储读写、MAC 地址读取 |
| [RNG](./references/rng.md) | 随机数生成器初始化、数据填充 |

### 系统工具

| 文档 | 内容 |
|------|------|
| [Blog](./references/blog.md) | 日志框架、DEBUG/INFO/WARN/ERROR |
| [CLI](./references/cli.md) | 命令行接口、静态/动态命令注册 |
| [EasyFlash](./references/easyflash.md) | KV 存储、ENV 环境变量 |
| [Utils](./references/utils.md) | CRC32/MD5/SHA256/Hex/Base64 工具函数 |
| [cJSON](./references/cjson.md) | JSON 解析/构建、数组/对象遍历 |

### 无线通信

| 文档 | 内容 |
|------|------|
| [Wi-Fi](./references/wifi.md) | Wi-Fi STA/AP 连接、扫描、功耗管理 |
| [BLE](./references/ble.md) | BLE 广播/扫描/连接、GATT 服务/通知 |
| [BLE Mesh](./references/ble-mesh.md) | BLE Mesh 配网、模型消息 |
| [LwIP Socket](./references/lwip.md) | TCP/UDP Socket、域名解析 |
| [MQTT](./references/mqtt.md) | MQTT 客户端、QoS 0/1/2、LWT |
| [HTTP](./references/http.md) | HTTPD/HTTPC GET/POST 请求 |
| [OTA](./references/ota.md) | HTTP/HTTPS OTA 固件升级 |
| [Wi-Fi HOSAL](./references/wifi-hosal.md) | Wi-Fi RF/功耗抽象层 |

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
