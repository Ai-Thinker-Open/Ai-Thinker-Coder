# BL616/BL618 系列模组开发指南

## 安信可 Ai-M61/Ai-M62 系列模组（BL616/BL618）

本 skill 为安信可科技 **BL616/BL618 系列 Wi-Fi 6 + BLE 5.0 模组**提供完整的开发指南。

---

## 快速开始

### 1. 安装工具链

```bash
# Windows MSYS2
pacman -S mingw-w64-x86_64-gcc python3 pip
pip install pyserial

# Linux/WSL
sudo apt install build-essential git python3 python3-pip
pip3 install pyserial
```

### 2. 克隆 SDK

```bash
git clone https://github.com/bouffalolab/bouffalo_sdk.git
cd bouffalo_sdk
git checkout v1.0.0
```

### 3. 编译固件

```bash
cd bouffalo_sdk/examples/helloworld
make CHIP=bl616 BOARD=bl616dk -j8
```

### 4. 烧录固件

```bash
make flash CHIP=bl616 BOARD=bl616dk COMX=/dev/ttyUSB0 BAUDRATE=2000000
```

---

## 芯片特性

| 特性 | BL616 | BL618 |
|------|-------|-------|
| CPU | 32-bit RISC-V, 320MHz | 同左 |
| Wi-Fi | 802.11a/b/g/n/ac/ax | 同左 |
| BLE | 5.0 + Mesh | 同左 |
| GPIO | 31 pins | 35 pins |
| UART | 2 通道 | 同左 |
| SPI | 1 通道 | 同左 |
| I2C | 1 通道 | 同左 |
| PWM | 5 通道 | 同左 |
| ADC | 12-bit SAR | 同左 |
| DMA | 8 通道 | 同左 |

---

## SKILL 目录结构

```
bl616_618/
├── SKILL.md              # 主入口（环境搭建、编程指南）
└── references/           # API 参考文档
    ├── gpio.md
    ├── uart.md
    ├── i2c.md
    ├── spi.md
    ├── timer.md
    ├── pwm.md
    ├── dma.md
    ├── adc.md
    ├── dac.md
    ├── flash.md
    ├── watchdog.md
    ├── rtc.md
    ├── rng.md
    └── efuse.md
```

---

## 驱动架构

BL616/BL618 使用 **LHAL (Low Level Hardware Abstraction Layer)** 驱动框架：

```c
#include "bflb_gpio.h"
#include "bflb_uart.h"

// 获取设备句柄
struct bflb_device_s *gpio = bflb_device_get_by_name("gpio");

// 初始化 GPIO
bflb_gpio_init(gpio, GPIO_PIN_0, GPIO_MODE_OUTPUT_PP, GPIO_PULL_UP);

// 操作 GPIO
bflb_gpio_set(gpio, GPIO_PIN_0);
```

---

## FreeRTOS 编程要点

BL616/BL618 的 FreeRTOS 应用注意事项：

1. **不需要调用 `vTaskStartScheduler()`** — 调度器由系统自动启动
2. **主循环必须调用 `vTaskDelay`** — 否则系统无法调度其他任务
3. **延时统一使用 `vTaskDelay(pdMS_TO_TICKS(ms))`**

```c
void my_task(void *param)
{
    while (1) {
        // 业务逻辑
        bflb_gpio_toggle(gpio, GPIO_PIN_0);
        vTaskDelay(pdMS_TO_TICKS(500));  // 必须延时
    }
}

void app_main(void)
{
    xTaskCreate(my_task, "blink", 512, NULL, 5, NULL);
    // 不需要 vTaskStartScheduler()
}
```

---

## 芯片对比

| 项目 | BL602 | BL616/BL618 |
|------|-------|-------------|
| Wi-Fi | 802.11b/g/n | **Wi-Fi 6** (ax) |
| BLE | 5.0 | 5.0 + **Mesh** |
| CPU 频率 | 40MHz | **320MHz** |
| GLB_BASE | 0x40000000 | **0x20000000** |
| GPIO 布局 | 每 2 引脚共享寄存器 | **每引脚独立寄存器** |
| 驱动框架 | HOSAL | **LHAL** |

---

## 资源链接

| 资源 | 链接 |
|-----|------|
| 官方文档站 | https://dev.bouffalolab.com/ |
| Bouffalo SDK | https://github.com/bouffalolab/bouffalo_sdk |
| BL616 数据手册 | https://dev.bouffalolab.com/media/doc/bl616/datasheet |
| BL618 数据手册 | https://dev.bouffalolab.com/media/doc/bl618/datasheet |
