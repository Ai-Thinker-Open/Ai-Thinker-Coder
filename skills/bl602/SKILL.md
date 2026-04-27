---
name: Ai-Thinker-Coder-bl602
description: 安信可 Ai-WB2 系列模组开发指南 (BL602 芯片平台) - Wi-Fi 4 + BLE 5.0 模组，支持二次开发，完整覆盖 GPIO、UART、DMA、定时器、PWM、ADC、I2C、SPI、WiFi、BLE 等外设编程，以及 MQTT、HTTP 等网络协议开发。使用 /skill Ai-Thinker-Coder 加载主入口。
category: hardware
tags:
  - 安信可
  - Ai-WB2
  - BL602
  - WiFi
  - BLE
  - 物联网
  - Bouffalolab
  - 二次开发
version: 1.0.0
author: 安信可科技 & Hermes Agent
license: MIT-0
metadata:
  chip: BL602
  module_family: Ai-WB2
  protocols: [Wi-Fi 4, BLE 5.0]
---

# 安信可 Ai-WB2 系列开发指南 (BL602)

## 编程范式说明

> **重要**：除非用户明确要求采用**寄存器级（bare metal）编程**，否则本 skill 提供的代码示例均使用 **HOSAL (Hardware Operating System Abstraction Layer) 库**进行开发，循环逻辑通过 **FreeRTOS 任务**实现。

**默认编程方式**（HOSAL + FreeRTOS）：
- 外设初始化、配置：调用 HOSAL API
- 业务逻辑循环：放在 FreeRTOS 任务中（`xTaskCreate`）
- 延时：`vTaskDelay`（非 `HAL_Delay`）

**BL602 FreeRTOS 特别说明**：
- **不需要调用 `vTaskStartScheduler()`**，系统会在 `main()` 中自动启动调度器
- `main()` 函数会执行初始化，然后进入主循环（裸机模式）或交由 RTOS 调度
- **主循环如果没有任何业务逻辑，必须调用 `vTaskDelay` 让出 CPU**，否则任务无法切换，系统表现为死机
- 正确的做法：`while(1) { 处理业务; vTaskDelay(pdMS_TO_TICKS(100)); }`

**寄存器级编程**（当用户明确要求时）：
- 直接操作 `*(volatile uint32_t *)addr` 访问外设寄存器
- 所有时序和配置需自行控制

---

## 产品概述

**Ai-WB2** 系列是安信可科技开发的 Wi-Fi & BLE 双模模组，基于 **BL602 芯片** (Bouffalolab)，支持：
- Wi-Fi 802.11b/g/n，20MHz 带宽，最高 72.2 Mbps
- Bluetooth Low Energy 5.0 + Bluetooth Mesh
- 32-bit RISC CPU (276KB RAM)
- 多种休眠模式，深度睡眠电流 12μA

### 选型表

| 型号 | 封装 | Flash | RAM | 天线 | 特点 |
|-----|------|-------|-----|------|-----|
| Ai-WB2-01F | SMD | 2MB | 276KB | 板载 PCB | 兼容 ESP8285 |
| Ai-WB2-01M | SMD | 2MB | 276KB | 邮票孔 | - |
| Ai-WB2-01S | SMD | 2MB | 276KB | IPEX | - |
| Ai-WB2-05W | SMD | 4MB | 276KB | 板载 PCB | - |
| Ai-WB2-07S | SMD | 4MB | 276KB | IPEX | - |
| Ai-WB2-12F | SMD | 4MB | 276KB | 板载 PCB | 高性价比 |
| Ai-WB2-12S | SMD | 4MB | 276KB | IPEX | - |
| Ai-WB2-13 | SMD | 4MB | 276KB | 陶瓷天线 | - |
| Ai-WB2-13U | SMD | 4MB | 276KB | USB | 带 USB 口 |
| Ai-WB2-32S | SMD | 4MB | 276KB | IPEX | - |
| Ai-WB2-M1 | SMD | 2MB | 276KB | - | - |
| Ai-WB2-M1-I | SMD | 2MB | 276KB | IPEX | - |

**选型表下载**: https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/ai-wb2_selection_table.xlsx

---

## 开发环境配置（必读）

> **重要提醒**：在编写任何代码之前，必须先完成环境配置。**工具链安装和 SDK 克隆是编程的前置条件**，否则无法编译和烧录固件。

### 第一步：判断当前平台

在开始之前，先确认当前操作系统：

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

# 安装编译工具链（包含 riscv64-unknown-elf-gcc）
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

# 安装 Python 依赖（烧录需要）
pip3 install pyserial

# 添加串口权限（可选，针对真实硬件）
sudo usermod -a -G dialout $USER
```

**验证安装**：

```bash
riscv64-unknown-elf-gcc -v    # 或 gcc -v，BL602 SDK 在 Linux 下使用系统 gcc
python3 --version              # 应输出版本号
pip3 show pyserial            # 应显示 pyserial 信息
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

工具链安装完成后，所有平台执行相同的 SDK 获取步骤：

```bash
# 选择合适的目录存放项目（如 ~/projects 或 D:\workspace）
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-WB2.git
cd Ai-Thinker-WB2

# 必须执行：初始化子模块（否则编译缺少关键文件）
git submodule update --init --recursive
```

> **国内用户如果 GitHub 访问不稳定**，可以换用 Gitee 镜像：
> ```bash
> git clone https://gitee.com/aithinker_open/Ai-Thinker-WB2.git
> cd Ai-Thinker-WB2
> git submodule update --init --recursive
> ```

**Linux/WSL 额外步骤**：设置工具链权限

```bash
cd toolchain/riscv/Linux/
bash chmod755.sh
```

### 第四步：编译验证

> **Windows 用户**：必须在「MSYS2 MINGW64」终端中执行编译，**不要使用 CMD、PowerShell 或普通的 MSYS2 终端**。

```bash
cd applications/get-started/helloworld
make -j8
```

编译成功会在 `build/out/` 目录生成固件文件。

执行 `make flash`（无论烧录是否成功）后，会在以下路径生成合并固件：

```
Ai-Thinker-WB2/tools/flash_tool/chips/bl602/img_create_iot/whole_flash_data.bin
```

> 这就是当前版本的完整合并固件，包含所有分区数据。用户需要提供固件时，将此文件按要求重命名后即可交付。

### 第五步：烧录验证

参见下方「固件烧录」章节。烧录成功且模组串口有输出才算环境配置完成。

---

## BL602 GPIO 寄存器编程

**前置条件**：已完成上述「开发环境配置」第二步和第三步。

**GPIO 基础地址**:
- GLB_BASE = 0x40000000
- GPIO配置寄存器: GLB_BASE + 0x100 + (pin/2)*4
- GPIO输出值: GLB_BASE + 0x188
- GPIO输出使能: GLB_BASE + 0x190

**每2个GPIO共用一个32bit寄存器，位域布局**:

偶数引脚 (0,2,4...):
```
[0]     IE        (1=输入, 0=输出)
[1]     SMT
[3:2]   DRV
[4]     PU
[5]     PD
[11:8]  FUNC_SEL  (11=GPIO模式)
```

奇数引脚 (1,3,5...): 高16bits相同偏移

**配置步骤**:
1. 清除IE位 (0=输出模式)
2. 设置FUNC_SEL=11 (GPIO模式)
3. 设置OUTPUT_EN寄存器对应位=1

**CPU频率**: 40MHz (默认,非120MHz)

**示例代码**:
```c
#define GLB_BASE  0x40000000
#define GLB_REG(off) (*(volatile uint32_t *)(GLB_BASE + off))

static void gpio_set_output(uint8_t pin) {
    uint32_t reg_off = 0x100 + (pin/2)*4;
    uint32_t tmp = GLB_REG(reg_off);
    
    if (pin % 2 == 0) {
        tmp &= ~(1 << 0);           // 清除IE
        tmp &= ~(0xF << 8);         // 清除FUNC_SEL
        tmp |= (11 << 8);           // 设置为GPIO模式
    } else {
        tmp &= ~(1 << 16);          // 清除IE
        tmp &= ~(0xF << 24);        // 清除FUNC_SEL
        tmp |= (11 << 24);          // 设置为GPIO模式
    }
    GLB_REG(reg_off) = tmp;
    GLB_REG(0x190) |= (1 << pin);  // 使能输出
}

static void gpio_write(uint8_t pin, uint8_t val) {
    if (val) GLB_REG(0x188) |= (1 << pin);
    else     GLB_REG(0x188) &= ~(1 << pin);
}
```

**项目结构**: `applications/get-started/<project>/<project>/main.c`

---

## 固件烧录

### 烧录工具

| 工具 | 下载 | 说明 |
|-----|------|-----|
| BL602 Flash Download Tool | [点击下载](https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/bl602_flash_download_tool.zip) | 官方工具 |
| 开发板专用工具(含GUI) | [点击下载](https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/v1.7.4-release.zip) | 带图形界面 |

### 烧录步骤

**第一步：列出可用串口供用户选择**

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

常见的串口芯片对应：
| 芯片 | Windows 串口名 | Linux 设备 |
|------|---------------|-----------|
| CH340 | COM3/COM4 等 | /dev/ttyUSB0 |
| CP2102 | COM5/COM6 等 | /dev/ttyUSB0 |
| FTDI | COM10+ | /dev/ttyUSB0 |

**第二步：进入烧录模式**

将模组的 **BOOT 引脚拉低**，然后复位（重新上电或拉低 RST）。

**第三步：烧录**

```bash
# 使用命令行烧录（Linux/macOS/MSYS2）
# 将 /dev/ttyUSB0 替换为实际串口名
make flash p=/dev/ttyUSB0 b=921600
```

烧录成功后，固件文件 `whole_flash_data.bin` 会在以下路径生成：
```
Ai-Thinker-WB2/tools/flash_tool/chips/bl602/img_create_iot/whole_flash_data.bin
```

**详细教程**: https://blog.csdn.net/Boantong_/article/details/125781602

---

## AT 指令开发

### AT 固件

| 固件号 | 版本 | 说明 |
|-------|------|-----|
| 2939 | V4.18_P23.2.1 | Combo-AT 中间件常规固件 |
| 2328 | V4.18_P23.1.0 | Combo-AT 中间件 |
| 1923 | V4.18_P1.4.4 | Combo-AT V2 |
| 2103 | V2.3.7 | AT 固件 |

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

### AT 固件更新日志

文档: `docs/zh/wifi/wb2/fw_server.md`（本 skill 仓库内文档路径）

---

## 二次开发 - 外设编程

### GPIO

```c
#include "hal_gpio.h"

// 初始化 GPIO
gpio_init_t gpio_init = {
    .port = GPIO_PORT_A,
    .pin = 0,
    .mode = GPIO_MODE_OUTPUT_PP,
    .pull = GPIO_PULLUP_PULLDOWN,
    .strength = GPIO_STRENGTH_MEDIUM,
};
gpio_init(&gpio_init);

// 写 GPIO
gpio_set_pin(GPIO_PORT_A, 0, 1);  // 置高
gpio_set_pin(GPIO_PORT_A, 0, 0);  // 置低

// 读 GPIO
uint8_t value = gpio_get_pin(GPIO_PORT_A, 0);

// 切换
gpio_toggle_pin(GPIO_PORT_A, 0);
```

### UART

```c
#include "hal_uart.h"

uart_config_t uart_config = {
    .baud_rate = 115200,
    .data_bits = UART_DATA_BITS_8,
    .stop_bits = UART_STOP_BITS_1,
    .parity = UART_PARITY_NONE,
    .flow_control = UART_FLOW_CONTROL_NONE,
    .rx_buffer_size = 256,
    .tx_buffer_size = 256,
};

uart_init(UART_ID_0, &uart_config);

// 发送数据
uint8_t tx_data[] = "Hello\r\n";
uart_send(UART_ID_0, tx_data, sizeof(tx_data));

// 接收数据 (中断方式)
void uart0_irq_handler(void)
{
    uint8_t data;
    if (uart_receive(UART_ID_0, &data, 1) > 0) {
        // 处理数据
    }
}
```

### DMA

```c
#include "hal_dma.h"

dma_config_t dma_config = {
    .channel = DMA_CHANNEL_0,
    .src_addr = (uint32_t)src_buffer,
    .dst_addr = (uint32_t)dst_buffer,
    .transfer_mode = DMA_TRANSFER_MODE_MEM_TO_MEM,
    .data_width = DMA_DATA_WIDTH_8BIT,
    .block_size = 256,
};

dma_init(DMA_CHANNEL_0, &dma_config);
dma_start(DMA_CHANNEL_0);

// 等待完成
while (!dma_transfer_done(DMA_CHANNEL_0));
```

### 定时器

```c
#include "hal_timer.h"

timer_config_t timer_config = {
    .timer_id = TIMER_ID_0,
    .mode = TIMER_MODE_PERIODIC,
    .period = 1000,  // 1ms
    .callback = timer0_callback,
};

timer_init(&timer_config);
timer_start(TIMER_ID_0);

void timer0_callback(void)
{
    // 定时处理
}
```

### PWM

```c
#include "hal_pwm.h"

pwm_config_t pwm_config = {
    .pwm_id = PWM_ID_0,
    .channel = 0,
    .frequency = 1000,  // 1kHz
    .duty_cycle = 50,  // 50%
    .pin = GPIO_PIN_0,
};

pwm_init(&pwm_config);
pwm_start(PWM_ID_0);

// 改变占空比
pwm_set_duty_cycle(PWM_ID_0, 75);  // 75%
```

### ADC

```c
#include "hal_adc.h"

adc_config_t adc_config = {
    .adc_id = ADC_ID_0,
    .channel = ADC_CHANNEL_0,
    .resolution = ADC_RESOLUTION_12BIT,
    .sample_rate = ADC_SAMPLE_RATE_1MSPS,
};

adc_init(&adc_config);

uint16_t adc_value;
adc_read(ADC_ID_0, ADC_CHANNEL_0, &adc_value);
// adc_value 范围 0-4095 (12bit)
```

### I2C

```c
#include "hal_i2c.h"

i2c_config_t i2c_config = {
    .i2c_id = I2C_ID_0,
    .mode = I2C_MODE_MASTER,
    .speed = I2C_SPEED_400K,
    .scl_pin = GPIO_PIN_1,
    .sda_pin = GPIO_PIN_2,
};

i2c_init(&i2c_config);

// 写寄存器
uint8_t write_data[2] = {reg_addr, value};
i2c_master_write(I2C_ID_0, device_addr, write_data, 2);

// 读寄存器
uint8_t read_data[2];
i2c_master_read(I2C_ID_0, device_addr, read_data, 2);
```

### SPI

```c
#include "hal_spi.h"

spi_config_t spi_config = {
    .spi_id = SPI_ID_0,
    .mode = SPI_MODE_MASTER,
    .clock_freq = 1000000,  // 1MHz
    .clock_polarity = SPI_CPOL_0,
    .clock_phase = SPI_CPHA_0,
    .data_width = SPI_DATA_WIDTH_8BIT,
    .cs_pin = GPIO_PIN_10,
};

spi_init(&spi_config);

// 发送接收
uint8_t tx_data[] = {0x01, 0x02, 0x03};
uint8_t rx_data[3];
spi_master_transfer(SPI_ID_0, tx_data, rx_data, 3);
```

---

## FreeRTOS 任务开发

### 任务创建示例

BL602 的 FreeRTOS 应用通常在 `main()` 中完成外设初始化后，创建业务任务。

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

    // BL602 不需要手动调用 vTaskStartScheduler()
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
// vTaskDelete(task_handle);            // 删除指定任务

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

wifi_conf_t wifi_conf = {
    .country_code = "CN",
};

wifi_manager_init(&wifi_conf);

// 连接 Wi-Fi
int ret = wifi_sta_connect("SSID", "PASSWORD");
if (ret == 0) {
    printf("WiFi connected\r\n");
    
    // 获取 IP
    wifi_sta_info_t info;
    wifi_sta_get_info(&info);
    printf("IP: %s\r\n", info.ipaddr);
}
```

### SoftAP 模式

```c
#include "wifi_mgmr.h"

// 创建热点
wifi_softap_config_t ap_config = {
    .ssid = "Ai-WB2-Setup",
    .password = "12345678",
    .channel = 6,
    .authmode = WIFI_AUTH_WPA2_PSK,
};

int ret = wifi_softap_start(&ap_config);
if (ret == 0) {
    printf("AP started\r\n");
}
```

### MQTT

```c
#include "mqtt_client.h"

mqtt_client_config_t mqtt_config = {
    .broker_url = "mqtt://broker.emqx.io:1883",
    .client_id = "ai-wb2-client",
    .username = "user",
    .password = "pass",
    .publish_topic = "/aiwb2/pub",
    .subscribe_topic = "/aiwb2/sub",
};

mqtt_client_handle_t mqtt = mqtt_client_new(&mqtt_config);

// 订阅回调
void mqtt_subscribe_callback(char *topic, uint8_t *data, uint32_t len)
{
    printf("Received on %s: %.*s\r\n", topic, len, data);
}

mqtt_client_subscribe(mqtt, "/aiwb2/sub", mqtt_subscribe_callback);

// 发布消息
mqtt_client_publish(mqtt, "/aiwb2/pub", (uint8_t *)"Hello", 5, 0);
```

### HTTP

```c
#include "http_client.h"

http_client_config_t http_config = {
    .url = "http://httpbin.org/get",
    .method = HTTP_METHOD_GET,
};

http_client_handle_t http = http_client_new(&http_config);

int ret = http_client_perform(http);
if (ret == 0) {
    char *response = http_client_get_response_body(http);
    printf("Response: %s\r\n", response);
}

http_client_free(http);
```

---

## 二次开发 - BLE 编程

### BLE 广播

```c
#include "ble_lib.h"

ble_gap_adv_params_t adv_params = {
    .adv_type = BLE_GAP_ADV_TYPE_IND,
    .adv_interval = 160,  // 100ms
    .own_addr_type = BLE_ADDR_TYPE_PUBLIC,
};

ble_adv_data_set_t adv_data = {
    .name = "Ai-WB2",
    .flags = BLE_GAP_ADV_FLAG_LE_GENERAL_DISC,
};

ble_stack_init();
ble_gap_adv_start(&adv_params, &adv_data);
```

### BLE 连接

```c
#include "ble_lib.h"

// BLE 服务和特征值定义
ble_uuid_t service_uuid = BLE_UUID16_DECLARE(0xFFF0);
ble_uuid_t char_uuid = BLE_UUID16_DECLARE(0xFFF1);

ble_attr_db_t att_db[] = {
    {
        .uuid = service_uuid,
        .type = BLE_ATTR_TYPE_PRIMARY_SERVICE,
    },
    {
        .uuid = char_uuid,
        .type = BLE_ATTR_TYPE_CHARACTERISTIC,
        .properties = BLE_CHAR_PROP_READ | BLE_CHAR_PROP_WRITE | BLE_CHAR_PROP_NOTIFY,
    },
};

// 创建服务
ble_svc_gatt_init();
ble_svc_gatt_add_service(service_uuid, att_db, sizeof(att_db));
```

---

## API 参考（HOSAL 外设库）

> 以下 API 来自 `components/platform/hosal/include/` 目录下的头文件，均为 BL602 官方 HOSAL 接口。包含文件路径、类型定义和函数签名。

---

### GPIO (`hosal_gpio.h`)

**类型定义**
```c
typedef enum {
    ANALOG_MODE,               // 模拟模式
    INPUT_PULL_UP,             // 输入上拉
    INPUT_PULL_DOWN,           // 输入下拉
    INPUT_HIGH_IMPEDANCE,      // 高阻输入
    OUTPUT_PUSH_PULL,          // 推挽输出
    OUTPUT_OPEN_DRAIN_NO_PULL, // 开漏输出（无上拉）
    OUTPUT_OPEN_DRAIN_PULL_UP, // 开漏输出（内置上拉）
    OUTPUT_OPEN_DRAIN_AF,      // 开漏复用功能
    OUTPUT_PUSH_PULL_AF,       // 推挽复用功能
} hosal_gpio_config_t;

typedef enum {
    HOSAL_IRQ_TRIG_NEG_PULSE,  // 下降沿脉冲触发
    HOSAL_IRQ_TRIG_POS_PULSE,  // 上升沿脉冲触发
    HOSAL_IRQ_TRIG_NEG_LEVEL,  // 下降沿电平触发（32k 3T）
    HOSAL_IRQ_TRIG_POS_LEVEL,  // 上升沿电平触发（32k 3T）
} hosal_gpio_irq_trigger_t;

typedef void (*hosal_gpio_irq_handler_t)(void *arg);

typedef struct {
    uint8_t        port;         // GPIO 端口
    hosal_gpio_config_t  config; // GPIO 配置模式
    void          *priv;         // 私有数据
} hosal_gpio_dev_t;
```

**函数接口**

| 函数 | 说明 |
|------|------|
| `int hosal_gpio_init(hosal_gpio_dev_t *gpio)` | 初始化 GPIO |
| `int hosal_gpio_output_set(hosal_gpio_dev_t *gpio, uint8_t value)` | 设置输出电平（0=低，>0=高） |
| `int hosal_gpio_input_get(hosal_gpio_dev_t *gpio, uint8_t *value)` | 读取输入电平 |
| `int hosal_gpio_irq_set(hosal_gpio_dev_t *gpio, hosal_gpio_irq_trigger_t trigger, hosal_gpio_irq_handler_t handler, void *arg)` | 配置中断触发 |
| `int hosal_gpio_irq_mask(hosal_gpio_dev_t *gpio, uint8_t mask)` | 屏蔽/使能中断（0=使能，1=屏蔽） |
| `int hosal_gpio_finalize(hosal_gpio_dev_t *gpio)` | 释放 GPIO |

---

### UART (`hosal_uart.h`)

**类型定义**
```c
typedef enum {
    HOSAL_DATA_WIDTH_5BIT,
    HOSAL_DATA_WIDTH_6BIT,
    HOSAL_DATA_WIDTH_7BIT,
    HOSAL_DATA_WIDTH_8BIT,  // 常用
    HOSAL_DATA_WIDTH_9BIT
} hosal_uart_data_width_t;

typedef enum {
    HOSAL_STOP_BITS_1,      // 1 位停止位（常用）
    HOSAL_STOP_BITS_1_5,    // 1.5 位停止位
    HOSAL_STOP_BITS_2       // 2 位停止位
} hosal_uart_stop_bits_t;

typedef enum {
    HOSAL_FLOW_CONTROL_DISABLED, // 无流控（常用）
    HOSAL_FLOW_CONTROL_CTS,
    HOSAL_FLOW_CONTROL_RTS,
    HOSAL_FLOW_CONTROL_CTS_RTS
} hosal_uart_flow_control_t;

typedef enum {
    HOSAL_NO_PARITY,        // 无校验（常用）
    HOSAL_ODD_PARITY,      // 奇校验
    HOSAL_EVEN_PARITY      // 偶校验
} hosal_uart_parity_t;

typedef enum {
    HOSAL_UART_MODE_POLL,      // 轮询模式（默认）
    HOSAL_UART_MODE_INT_TX,    // TX 中断模式
    HOSAL_UART_MODE_INT_RX,    // RX 中断模式
    HOSAL_UART_MODE_INT,       // TX+RX 中断模式
} hosal_uart_mode_t;

typedef struct {
    uint8_t                   uart_id;        // UART ID (0/1/2)
    uint8_t                   tx_pin;         // TX 引脚
    uint8_t                   rx_pin;         // RX 引脚
    uint8_t                   cts_pin;       // CTS 引脚（255=不用）
    uint8_t                   rts_pin;       // RTS 引脚（255=不用）
    uint32_t                  baud_rate;     // 波特率（115200/9600 等）
    hosal_uart_data_width_t   data_width;    // 数据位宽
    hosal_uart_parity_t       parity;        // 校验位
    hosal_uart_stop_bits_t    stop_bits;     // 停止位
    hosal_uart_flow_control_t flow_control; // 流控制
    hosal_uart_mode_t         mode;          // 模式
} hosal_uart_config_t;

typedef struct {
    uint8_t       port;
    hosal_uart_config_t config;
    hosal_uart_callback_t tx_cb;            // TX 回调
    void *p_txarg;
    hosal_uart_callback_t rx_cb;            // RX 回调
    void *p_rxarg;
    hosal_uart_callback_t txdma_cb;         // TX DMA 回调
    void *p_txdma_arg;
    hosal_uart_callback_t rxdma_cb;         // RX DMA 回调
    void *p_rxdma_arg;
    hosal_dma_chan_t dma_tx_chan;           // DMA TX 通道
    hosal_dma_chan_t dma_rx_chan;           // DMA RX 通道
    void         *priv;
} hosal_uart_dev_t;
```

**函数接口**

| 函数 | 说明 |
|------|------|
| `int hosal_uart_init(hosal_uart_dev_t *uart)` | 初始化 UART |
| `int hosal_uart_init_only_tx(hosal_uart_dev_t *uart)` | 仅初始化 TX（单向） |
| `int hosal_uart_send(hosal_uart_dev_t *uart, const void *txbuf, uint32_t size)` | 轮询发送数据，返回实际发送字节数 |
| `int hosal_uart_receive(hosal_uart_dev_t *uart, void *data, uint32_t expect_size)` | 轮询接收数据，返回实际接收字节数 |
| `int hosal_uart_ioctl(hosal_uart_dev_t *uart, int ctl, void *p_arg)` | IO 控制（设置波特率、数据位、停止位、校验、流控等） |
| `int hosal_uart_callback_set(hosal_uart_dev_t *uart, int callback_type, hosal_uart_callback_t pfn, void *arg)` | 设置中断回调 |
| `int hosal_uart_finalize(hosal_uart_dev_t *uart)` | 释放 UART |

**`hosal_uart_ioctl` 控制命令**

| 命令 | 说明 |
|------|------|
| `HOSAL_UART_BAUD_SET` / `_GET` | 设置/获取波特率，p_arg 为 `uint32_t *` |
| `HOSAL_UART_DATA_WIDTH_SET` / `_GET` | 设置/获取数据位宽，p_arg 为 `hosal_uart_data_width_t *` |
| `HOSAL_UART_STOP_BITS_SET` / `_GET` | 设置/获取停止位，p_arg 为 `hosal_uart_stop_bits_t *` |
| `HOSAL_UART_PARITY_SET` / `_GET` | 设置/获取校验位，p_arg 为 `hosal_uart_parity_t *` |
| `HOSAL_UART_MODE_SET` / `_GET` | 设置/获取模式，p_arg 为 `hosal_uart_mode_t *` |
| `HOSAL_UART_FLUSH` | 等待发送完成 |
| `HOSAL_UART_DMA_TX_START` / `_RX_START` | 启动 DMA 传输，p_arg 为 `hosal_uart_dma_cfg_t *` |

**宏**

```c
// 声明 UART 配置（快速定义）
HOSAL_UART_CFG_DECL(cfg, id, tx_pin, rx_pin, baud);
// 例: HOSAL_UART_CFG_DECL(my_uart_cfg, 0, 16, 7, 115200);

// 声明 UART 设备
HOSAL_UART_DEV_DECL(my_uart, 0, 16, 7, 115200);
```

---

### I2C (`hosal_i2c.h`)

**类型定义**
```c
#define HOSAL_WAIT_FOREVER 0xFFFFFFFFU  // 无限等待

#define HOSAL_I2C_MODE_MASTER 1          // 主机模式
#define HOSAL_I2C_MODE_SLAVE  2          // 从机模式

#define HOSAL_I2C_ADDRESS_WIDTH_7BIT  0  // 7 位地址（常用）
#define HOSAL_I2C_ADDRESS_WIDTH_10BIT 1  // 10 位地址

typedef struct {
    uint32_t address_width;  // 地址宽度：7bit / 10bit
    uint32_t freq;           // I2C 频率（Hz），如 400000 = 400kHz
    uint8_t  scl;            // SCL 引脚
    uint8_t  sda;            // SDA 引脚
    uint8_t  mode;           // 主机/从机模式
} hosal_i2c_config_t;

typedef struct {
    uint8_t       port;       // I2C 端口号 (0/1)
    hosal_i2c_config_t  config;
    void         *priv;
} hosal_i2c_dev_t;
```

**函数接口**

| 函数 | 说明 |
|------|------|
| `int hosal_i2c_init(hosal_i2c_dev_t *i2c)` | 初始化 I2C |
| `int hosal_i2c_master_send(hosal_i2c_dev_t *i2c, uint16_t dev_addr, const uint8_t *data, uint16_t size, uint32_t timeout)` | 主机发送 |
| `int hosal_i2c_master_recv(hosal_i2c_dev_t *i2c, uint16_t dev_addr, uint8_t *data, uint16_t size, uint32_t timeout)` | 主机接收 |
| `int hosal_i2c_slave_send(hosal_i2c_dev_t *i2c, const uint8_t *data, uint16_t size, uint32_t timeout)` | 从机发送 |
| `int hosal_i2c_slave_recv(hosal_i2c_dev_t *i2c, uint8_t *data, uint16_t size, uint32_t timeout)` | 从机接收 |
| `int hosal_i2c_mem_write(hosal_i2c_dev_t *i2c, uint16_t dev_addr, uint32_t mem_addr, uint16_t mem_addr_size, const uint8_t *data, uint16_t size, uint32_t timeout)` | 内存写（带寄存器地址） |
| `int hosal_i2c_mem_read(hosal_i2c_dev_t *i2c, uint16_t dev_addr, uint32_t mem_addr, uint16_t mem_addr_size, uint8_t *data, uint16_t size, uint32_t timeout)` | 内存读（带寄存器地址） |
| `int hosal_i2c_finalize(hosal_i2c_dev_t *i2c)` | 释放 I2C |

---

### SPI (`hosal_spi.h`)

**类型定义**
```c
#define HOSAL_SPI_MODE_MASTER 0  // 主机模式
#define HOSAL_SPI_MODE_SLAVE  1  // 从机模式
#define HOSAL_WAIT_FOREVER  0xFFFFFFFFU

typedef void (*hosal_spi_irq_t)(void *parg);

typedef struct {
    uint8_t mode;           // 主机/从机模式
    uint8_t dma_enable;     // 是否启用 DMA (0=禁用)
    uint8_t polar_phase;    // 极性和相位 (CPOL=0/1, CPHA=0/1)
    uint32_t freq;          // 通信频率 Hz（如 1000000 = 1MHz）
    uint8_t pin_clk;        // CLK 引脚
    uint8_t pin_mosi;       // MOSI 引脚
    uint8_t pin_miso;       // MISO 引脚
} hosal_spi_config_t;

typedef struct {
    uint8_t port;
    hosal_spi_config_t  config;
    hosal_spi_irq_t cb;     // 中断回调
    void *p_arg;
    void *priv;
} hosal_spi_dev_t;
```

**函数接口**

| 函数 | 说明 |
|------|------|
| `int hosal_spi_init(hosal_spi_dev_t *spi)` | 初始化 SPI |
| `int hosal_spi_send(hosal_spi_dev_t *spi, const uint8_t *data, uint16_t size, uint32_t timeout)` | 仅发送 |
| `int hosal_spi_recv(hosal_spi_dev_t *spi, uint8_t *data, uint16_t size, uint32_t timeout)` | 仅接收 |
| `int hosal_spi_send_recv(hosal_spi_dev_t *spi, uint8_t *tx_data, uint8_t *rx_data, uint16_t size, uint32_t timeout)` | 发送并接收（常用） |
| `int hosal_spi_irq_callback_set(hosal_spi_dev_t *spi, hosal_spi_irq_t pfn, void *p_arg)` | 设置中断回调 |
| `int hosal_spi_set_cs(uint8_t pin, uint8_t value)` | 软件控制 CS 片选引脚（仅主机） |
| `int hosal_spi_finalize(hosal_spi_dev_t *spi)` | 释放 SPI |

---

### DMA (`hosal_dma.h`)

**类型定义**
```c
#define HOSAL_DMA_INT_TRANS_COMPLETE 0  // 传输完成中断
#define HOSAL_DMA_INT_TRANS_ERROR    1  // 传输错误中断

typedef void (*hosal_dma_irq_t)(void *p_arg, uint32_t flag);
typedef int hosal_dma_chan_t;           // DMA 通道号

typedef struct hosal_dma_dev {
    int max_chans;
    struct hosal_dma_chan *used_chan;
    void *priv;
} hosal_dma_dev_t;
```

**函数接口**

| 函数 | 说明 |
|------|------|
| `int hosal_dma_init(void)` | 初始化 DMA（全局） |
| `hosal_dma_chan_t hosal_dma_chan_request(int flag)` | 请求 DMA 通道，返回通道号 |
| `int hosal_dma_chan_release(hosal_dma_chan_t chan)` | 释放 DMA 通道 |
| `int hosal_dma_chan_start(hosal_dma_chan_t chan)` | 启动 DMA 传输 |
| `int hosal_dma_chan_stop(hosal_dma_chan_t chan)` | 停止 DMA 传输 |
| `int hosal_dma_irq_callback_set(hosal_dma_chan_t chan, hosal_dma_irq_t pfn, void *p_arg)` | 设置 DMA 中断回调 |
| `int hosal_dma_finalize(void)` | 释放 DMA |

---

### Timer (`hosal_timer.h`)

**类型定义**
```c
#define TIMER_RELOAD_PERIODIC 1  // 周期重复定时
#define TIMER_RELOAD_ONCE     2  // 单次定时

typedef void (*hosal_timer_cb_t)(void *arg);

typedef struct {
    uint32_t          period;      // 定时周期（微秒 us）
    uint8_t           reload_mode; // 重复模式
    hosal_timer_cb_t  cb;          // 定时回调函数
    void              *arg;         // 回调参数
} hosal_timer_config_t;

typedef struct {
    int8_t                port;   // 定时器端口号
    hosal_timer_config_t  config;
    void                  *priv;
} hosal_timer_dev_t;
```

**函数接口**

| 函数 | 说明 |
|------|------|
| `int hosal_timer_init(hosal_timer_dev_t *tim)` | 初始化定时器 |
| `int hosal_timer_start(hosal_timer_dev_t *tim)` | 启动定时器 |
| `void hosal_timer_stop(hosal_timer_dev_t *tim)` | 停止定时器 |
| `int hosal_timer_finalize(hosal_timer_dev_t *tim)` | 释放定时器 |

---

### PWM (`hosal_pwm.h`)

**类型定义**
```c
typedef struct {
    uint8_t    pin;        // PWM 引脚
    uint32_t   duty_cycle; // 占空比，范围 0~10000（对应 0%~100%）
    uint32_t   freq;       // 频率 Hz，最大 40MHz
} hosal_pwm_config_t;

typedef struct {
    uint8_t       port;         // PWM 端口
    hosal_pwm_config_t  config;
    void         *priv;
} hosal_pwm_dev_t;
```

**函数接口**

| 函数 | 说明 |
|------|------|
| `int hosal_pwm_init(hosal_pwm_dev_t *pwm)` | 初始化 PWM |
| `int hosal_pwm_start(hosal_pwm_dev_t *pwm)` | 启动 PWM 输出 |
| `int hosal_pwm_stop(hosal_pwm_dev_t *pwm)` | 停止 PWM 输出 |
| `int hosal_pwm_para_chg(hosal_pwm_dev_t *pwm, hosal_pwm_config_t para)` | 动态更改参数（频率+占空比） |
| `int hosal_pwm_freq_set(hosal_pwm_dev_t *pwm, uint32_t freq)` | 单独设置频率 |
| `int hosal_pwm_freq_get(hosal_pwm_dev_t *pwm, uint32_t *p_freq)` | 获取当前频率 |
| `int hosal_pwm_duty_set(hosal_pwm_dev_t *pwm, uint32_t duty)` | 单独设置占空比（0~10000） |
| `int hosal_pwm_duty_get(hosal_pwm_dev_t *pwm, uint32_t *p_duty)` | 获取当前占空比 |
| `int hosal_pwm_finalize(hosal_pwm_dev_t *pwm)` | 释放 PWM |

---

### ADC (`hosal_adc.h`)

**类型定义**
```c
#define HOSAL_WAIT_FOREVER 0xFFFFFFFFU

typedef enum {
    HOSAL_ADC_ONE_SHOT,   // 单次采样
    HOSAL_ADC_CONTINUE    // 连续采样
} hosal_adc_sample_mode_t;

typedef enum {
    HOSAL_ADC_INT_OV,      // 溢出错误
    HOSAL_ADC_INT_EOS,     // 采样结束
    HOSAL_ADC_INT_DMA_TRH, // DMA 传输半满
    HOSAL_ADC_INT_DMA_TRC, // DMA 传输完成
    HOSAL_ADC_INT_DMA_TRE, // DMA 传输错误
} hosal_adc_event_t;

typedef struct {
    uint32_t sampling_freq;      // 采样频率 Hz
    uint32_t pin;                // ADC 引脚
    hosal_adc_sample_mode_t mode; // 采样模式
    uint8_t  sample_resolution;  // 采样分辨率（位数）
} hosal_adc_config_t;

typedef void (*hosal_adc_cb_t)(hosal_adc_event_t event, void *data, uint32_t size);

typedef struct {
    uint8_t port;
    hosal_adc_config_t config;
    hosal_dma_chan_t dma_chan;
    hosal_adc_irq_t cb;
    void *p_arg;
    void *priv;
} hosal_adc_dev_t;
```

**函数接口**

| 函数 | 说明 |
|------|------|
| `int hosal_adc_init(hosal_adc_dev_t *adc)` | 初始化 ADC |
| `int hosal_adc_add_channel(hosal_adc_dev_t *adc, uint32_t channel)` | 添加采样通道 |
| `int hosal_adc_remove_channel(hosal_adc_dev_t *adc, uint32_t channel)` | 移除采样通道 |
| `int hosal_adc_add_reference_channel(hosal_adc_dev_t *adc, uint32_t refer_channel, float refer_voltage)` | 添加参考通道 |
| `int hosal_adc_remove_reference_channel(hosal_adc_dev_t *adc)` | 移除参考通道 |
| `int hosal_adc_value_get(hosal_adc_dev_t *adc, uint32_t channel, uint32_t timeout)` | 读取单次采样值（阻塞） |
| `int hosal_adc_tsen_value_get(hosal_adc_dev_t *adc)` | 读取内部温度传感器 |
| `int hosal_adc_sample_cb_reg(hosal_adc_dev_t *adc, hosal_adc_cb_t cb)` | 注册采样回调 |
| `int hosal_adc_start(hosal_adc_dev_t *adc, void *data, uint32_t size)` | 启动连续采样 |
| `int hosal_adc_stop(hosal_adc_dev_t *adc)` | 停止采样 |
| `int hosal_adc_finalize(hosal_adc_dev_t *adc)` | 释放 ADC |

---

### Flash (`hosal_flash.h`)

**类型定义**
```c
#define HOSAL_FLASH_FLAG_ADDR_0     0       // 使用分区表地址 0
#define HOSAL_FLASH_FLAG_ADDR_1     (1<<0)  // 使用分区表地址 1
#define HOSAL_FLASH_FLAG_BUSADDR    (1<<1)   // 使用总线物理地址

typedef struct {
    const char  *partition_description; // 分区名称
    uint32_t     partition_start_addr; // 分区起始地址
    uint32_t     partition_length;     // 分区长度
    uint32_t     partition_options;    // 选项
} hosal_logic_partition_t;

typedef struct hosal_flash_dev {
    void *flash_dev;
} hosal_flash_dev_t;
```

**函数接口**

| 函数 | 说明 |
|------|------|
| `hosal_flash_dev_t *hosal_flash_open(const char *name, unsigned int flags)` | 打开 Flash 分区，返回设备句柄 |
| `int hosal_flash_info_get(hosal_flash_dev_t *p_dev, hosal_logic_partition_t *partition)` | 获取分区信息 |
| `int hosal_flash_erase(hosal_flash_dev_t *p_dev, uint32_t off_set, uint32_t size)` | 擦除分区（按扇区） |
| `int hosal_flash_write(hosal_flash_dev_t *p_dev, uint32_t *off_set, const void *in_buf, uint32_t in_buf_size)` | 写入（不擦除，写入前需确保为 0xFF） |
| `int hosal_flash_erase_write(hosal_flash_dev_t *p_dev, uint32_t *off_set, const void *in_buf, uint32_t in_buf_size)` | 擦除并写入（常用） |
| `int hosal_flash_read(hosal_flash_dev_t *p_dev, uint32_t *off_set, void *out_buf, uint32_t out_buf_size)` | 读取 |
| `int hosal_flash_raw_read(void *buffer, uint32_t address, uint32_t length)` | 原始读取（物理地址） |
| `int hosal_flash_raw_write(void *buffer, uint32_t address, uint32_t length)` | 原始写入（物理地址，需先擦除） |
| `int hosal_flash_raw_erase(uint32_t start_addr, uint32_t length)` | 原始擦除（物理地址） |
| `int hosal_flash_close(hosal_flash_dev_t *p_dev)` | 关闭分区 |

> `hosal_flash_open` 的 `name` 参数为分区名称字符串，如 `"app"`、`"wifi"` 等，具体分区表定义在 `partition.csv` 文件中。

---

### Watchdog (`hosal_wdg.h`)

**类型定义**
```c
typedef struct {
    uint32_t timeout;  // 看门狗超时时间（毫秒 ms）
} hosal_wdg_config_t;

typedef struct {
    uint8_t       port;
    hosal_wdg_config_t  config;
    void         *priv;
} hosal_wdg_dev_t;
```

**函数接口**

| 函数 | 说明 |
|------|------|
| `int hosal_wdg_init(hosal_wdg_dev_t *wdg)` | 初始化看门狗 |
| `void hosal_wdg_reload(hosal_wdg_dev_t *wdg)` | 喂狗（重载计数器） |
| `int hosal_wdg_finalize(hosal_wdg_dev_t *wdg)` | 释放看门狗 |

---

### RTC (`hosal_rtc.h`)

**类型定义**
```c
#define HOSAL_RTC_FORMAT_DEC 1  // 十进制格式
#define HOSAL_RTC_FORMAT_BCD 2  // BCD 格式

typedef struct {
    uint8_t  sec;     // 秒（DEC: 0~59 / BCD: 0x00~0x59）
    uint8_t  min;     // 分（DEC: 0~59 / BCD: 0x00~0x59）
    uint8_t  hr;      // 时（DEC: 0~23 / BCD: 0x00~0x23）
    uint8_t  date;    // 日（DEC: 1~31 / BCD: 0x01~0x31）
    uint8_t  month;   // 月（DEC: 1~12 / BCD: 0x01~0x12）
    uint16_t year;    // 年（DEC: 0~9999 / BCD: 0x0000~0x9999）
} hosal_rtc_time_t;

typedef struct {
    uint8_t       port;
    hosal_rtc_config_t  config;
    void         *priv;
} hosal_rtc_dev_t;
```

**函数接口**

| 函数 | 说明 |
|------|------|
| `int hosal_rtc_init(hosal_rtc_dev_t *rtc)` | 初始化 RTC |
| `int hosal_rtc_set_time(hosal_rtc_dev_t *rtc, const hosal_rtc_time_t *time)` | 设置时间（struct 方式） |
| `int hosal_rtc_get_time(hosal_rtc_dev_t *rtc, hosal_rtc_time_t *time)` | 读取时间（struct 方式） |
| `int hosal_rtc_set_count(hosal_rtc_dev_t *rtc, uint64_t *time_stamp)` | 设置时间（时间戳方式） |
| `int hosal_rtc_get_count(hosal_rtc_dev_t *rtc, uint64_t *time_stamp)` | 读取时间（时间戳方式） |
| `int hosal_rtc_finalize(hosal_rtc_dev_t *rtc)` | 释放 RTC |

---

## 外设驱动移植

### SHT30 (温湿度传感器)

```c
#include "i2c_lib.h"

#define SHT30_ADDR 0x44

void sht30_init(void)
{
    uint8_t cmd[2] = {0x30, 0x93};  // 高精度模式
    i2c_master_write(I2C_ID_0, SHT30_ADDR, cmd, 2);
    vTaskDelay(pdMS_TO_TICKS(20));
}

int sht30_read(float *temp, float *humidity)
{
    uint8_t data[6];
    uint8_t cmd[2] = {0x24, 0x00};  // 读取命令
    i2c_master_write(I2C_ID_0, SHT30_ADDR, cmd, 2);
    vTaskDelay(pdMS_TO_TICKS(15));
    i2c_master_read(I2C_URL, SHT30_ADDR, data, 6);
    
    // 解析数据
    uint16_t temp_raw = (data[0] << 8) | data[1];
    uint16_t hum_raw = (data[3] << 8) | data[4];
    
    *temp = -45 + 175 * ((float)temp_raw / 65535);
    *humidity = 100 * ((float)hum_raw / 65535);
    
    return 0;
}
```

### WS2812B (RGB LED)

```c
#include "spi_lib.h"

#define WS2812_BITS 24  // 8bits per color

void ws2812_set_color(uint8_t r, uint8_t g, uint8_t b)
{
    uint8_t tx_data[WS2812_BITS];
    
    // GRB 顺序
    for (int i = 0; i < 8; i++) {
        tx_data[i] = (g & (0x80 >> i)) ? 0x7E : 0x10;  // Green
        tx_data[i + 8] = (r & (0x80 >> i)) ? 0x7E : 0x10;  // Red
        tx_data[i + 16] = (b & (0x80 >> i)) ? 0x7E : 0x10;  // Blue
    }
    
    spi_master_transfer(SPI_ID_0, tx_data, NULL, WS2812_BITS);
}
```

---

## 开发教程

### 入门篇

| 教程 | 链接 |
|-----|------|
| Ai-WB2 概述 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45158 |
| GPIO 使用 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45174 |
| UART 数据收发 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45176 |
| DMA 数据传输 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45184 |
| 定时器使用 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45185 |
| 看门狗使用 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45189 |
| PWM 使用 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45208 |

#### RGB LED Demo (纯寄存器编程)

**路径**: `applications/get-started/rgb_demo/`

**GPIO 映射** (Ai-WB2-12F-Kit):
| 颜色 | GPIO | 备注 |
|------|------|------|
| 红   | IO14 | |
| 绿   | IO17 | |
| 蓝   | IO3  | 主 LED |

**编译**:
```bash
cd applications/get-started/rgb_demo && make
```

**烧录**:
```bash
make flash p=/dev/ttyUSB0 b=921600
```

**四种闪烁模式**:
1. **呼吸灯** - 亮度平滑渐变,人眼舒适
2. **SOS 信号** - 三短三长三短求救编码
3. **七彩渐变** - 蓝→紫→红→青→绿→蓝
4. **快速闪烁** - 短促闪烁5次

**模式切换**: 淡入淡出过渡,循环执行

---

#### BL602 GPIO 寄存器详解 (经验证)

**重要**: BL602 的 GPIO 寄存器布局与常见 MCU 不同，必须按以下定义：

```c
/* BL602 寄存器地址 */
#define GLB_BASE                 0x40000000
#define GLB_GPIO_CFGCTL0        0x100   /* GPIO 0-1 配置寄存器 */
#define GLB_GPIO_OUTPUT_OFFSET   0x188   /* GPIO 输出值寄存器 */
#define GLB_GPIO_OUTPUT_EN_OFFSET 0x190  /* GPIO 输出使能寄存器 */

#define GLB_REG(off)  (*(volatile uint32_t *)(GLB_BASE + (off)))
```

**GPIO 配置寄存器 (0x100 起) 位域布局** (每2个GPIO共用一个32bit寄存器):

| GPIO 0 (低16bits) | 位置 | GPIO 1 (高16bits) | 位置 |
|-------------------|------|-------------------|------|
| IE (输入使能) | bit[0] | IE | bit[16] |
| SMT | bit[1] | SMT | bit[17] |
| DRV | bits[3:2] | DRV | bits[19:18] |
| PU | bit[4] | PU | bit[20] |
| PD | bit[5] | PD | bit[21] |
| **FUNC_SEL** | **bits[11:8]** | **FUNC_SEL** | **bits[27:24]** |

```c
/** GPIO 配置为输出模式 - 纯寄存器操作 (经验证正确) */
static void gpio_set_output(uint8_t pin)
{
    uint32_t reg_off;
    uint32_t tmp;
    uint8_t func_sel_pos;
    uint8_t ie_pos;
    
    reg_off = GLB_GPIO_CFGCTL0 + (pin / 2) * 4;
    tmp = GLB_REG(reg_off);
    
    if (pin % 2 == 0) {
        /* 偶引脚 (0, 2, 4...): FUNC_SEL at bits[11:8], IE at bit[0] */
        func_sel_pos = 8;
        ie_pos = 0;
    } else {
        /* 奇引脚 (1, 3, 5...): FUNC_SEL at bits[27:24], IE at bit[16] */
        func_sel_pos = 24;
        ie_pos = 16;
    }
    
    /* 清除IE位 (0=输出模式) */
    tmp &= ~(1 << ie_pos);
    /* 设置FUNC_SEL=11 (GPIO_FUN_GPIO) */
    tmp &= ~(0xF << func_sel_pos);
    tmp |= (11 << func_sel_pos);
    
    GLB_REG(reg_off) = tmp;
    
    /* 使能输出: 在OUTPUT_EN寄存器中设置对应位为1 */
    GLB_REG(GLB_GPIO_OUTPUT_EN_OFFSET) |= (1 << pin);
}

/** GPIO 输出控制 */
static void gpio_write(uint8_t pin, uint8_t value)
{
    if (value)
        GLB_REG(GLB_GPIO_OUTPUT_OFFSET) |= (1 << pin);
    else
        GLB_REG(GLB_GPIO_OUTPUT_OFFSET) &= ~(1 << pin);
}
```

**关键经验**:
1. **FUNC_SEL = 11** 是 GPIO 模式
2. **输出使能是独立寄存器** `0x190`，不是配置寄存器的一部分
3. **每2个引脚共用配置寄存器**，通过 `pin / 2` 索引
4. **偶引脚**: FUNC_SEL在bits[11:8], IE在bit[0]
5. **奇引脚**: FUNC_SEL在bits[27:24], IE在bit[16]
6. **CPU 频率 40MHz**（BL602默认值），1us ≈ 10 周期

**延时函数**:
```c
#define CPU_FREQ 40000000

static void delay_us(uint32_t us)
{
    volatile uint32_t count = us * (CPU_FREQ / 1000000 / 4);
    while (count--) { __asm__("nop"); }
}

static void delay_ms(uint32_t ms)
{
    delay_us(ms * 1000);
}
```

**参考源码**: `components/platform/soc/bl602/bl602_std/bl602_std/StdDriver/Src/bl602_glb.c` 第 1856-1935 行

### 中级篇

| 教程 | 链接 |
|-----|------|
| ADC 模数转换 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45179 |
| DAC 数模转换 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45211 |
| RTC 实时时钟 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45214 |
| I2C 通信 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45215 |
| SPI 通信 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45212 |
| SPI 与 WS2812B | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45228 |

### 网络篇

| 教程 | 链接 |
|-----|------|
| TCP 无线通讯 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45254 |
| UDP 无线通讯 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45260 |
| MQTT 协议连接 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45256 |
| HTTP 天气请求 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45238 |
| BLE 蓝牙通讯 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45268 |

### 外设移植篇

| 教程 | 链接 |
|-----|------|
| BH1750 光照传感器 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45213 |
| SHT30 温湿度传感器 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45241 |
| AHT20 温湿度传感器 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45242 |
| Modbus 485 RTU | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45244 |
| TM1637 NTP 时钟 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45246 |
| 舵机控制 SG90 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45247 |
| MPU6050 | https://bbs.ai-thinker.com/forum.php?mod=viewthread&tid=45302 |

---

## 常用资源链接

| 资源 | 链接 |
|-----|------|
| SDK 源码 | https://github.com/Ai-Thinker-Open/Ai-Thinker-WB2 |
| AT 指令集 (中文) | https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/combo%E6%A8%A1%E7%BB%84%E9%80%9A%E7%94%A8%E6%8C%87%E4%BB%A4_v4.18p_3.8.0.pdf |
| AT 指令集 (英文) | https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/Combo_AT_General_instructions_V4.18P_3.6.0_EN.pdf |
| 编程指南 | https://wb2-api-web.readthedocs.io/en/latest/docs/api-guides/index.html |
| 固件烧录工具 | https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/bl602_flash_download_tool.zip |
| 静态内存分析 | https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/bouffalo_parse_tool-win32.zip |
| 调试工具 | https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/ComAssistant_2.0.2.9.zip |
| 爱星云平台固件 | https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/media/WiFi/WB2/Firmware/wb2-aithings-illumination-v4.18_p0.0.1.zip |

---

## 芯片参考手册

| 文档 | 链接 |
|-----|------|
| BL602 Datasheet (中文) | https://dev.bouffalolab.com/media/doc/602/open/datasheet/zh/html/index.html |
| BL602 Datasheet (EN) | https://dev.bouffalolab.com/media/doc/602/open/datasheet/en/html/index.html |
| BL602 Reference Manual (中文) | https://dev.bouffalolab.com/media/doc/602/open/reference_manual/zh/html/index.html |
| BL602 Reference Manual (EN) | https://dev.bouffalolab.com/media/doc/602/open/reference_manual/en/html/index.html |

---

## FAQ 常见问题

**Q: 编译报错 "No such file or directory"**
```bash
# 重新初始化子模块
git submodule update --init --recursive
```

**Q: 烧录失败**
- 检查 BOOT 引脚是否拉低
- 检查串口驱动是否安装
- 检查串口选择是否正确

**Q: WiFi 连接失败**
- 检查 SSID 和密码是否正确
- 检查路由器是否支持 2.4G 频段
- 检查国家码设置

**Q: BLE 连接不稳定**
- 检查天线是否连接正确
- 减少障碍物和干扰源

**完整 FAQ**: https://aithinker.readthedocs.io/zh-cn/latest/docs/software-framework/WiFi/index.html#id2
