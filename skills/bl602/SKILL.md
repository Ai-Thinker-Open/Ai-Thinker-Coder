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

1. 将模组进入烧录模式（BOOT 引脚拉低，复位）
2. 打开烧录工具，选择对应串口
3. 选择固件文件
4. 设置烧录地址 0x0
5. 点击开始烧录

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
