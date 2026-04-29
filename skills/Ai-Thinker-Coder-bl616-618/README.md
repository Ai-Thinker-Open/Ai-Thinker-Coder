# BL616/BL618 Series Module Development Guide

## Ai-Thinker Ai-M61/Ai-M62 Series Modules (BL616/BL618)

This skill provides a complete development guide for Ai-Thinker **BL616/BL618 Series Wi-Fi 6 + BLE 5.0 modules**.

---

## Quick Start

### 1. Install Toolchain

```bash
# Windows MSYS2
pacman -S mingw-w64-x86_64-gcc python3 pip
pip install pyserial

# Linux/WSL
sudo apt install build-essential git python3 python3-pip
pip3 install pyserial
```

### 2. Clone SDK

```bash
git clone https://github.com/bouffalolab/bouffalo_sdk.git
cd bouffalo_sdk
git checkout v1.0.0
```

### 3. Build Firmware

```bash
cd bouffalo_sdk/examples/helloworld
make CHIP=bl616 BOARD=bl616dk -j8
```

### 4. Flash Firmware

```bash
make flash CHIP=bl616 BOARD=bl616dk COMX=/dev/ttyUSB0 BAUDRATE=2000000
```

---

## Chip Features

| Feature | BL616 | BL618 |
|------|-------|-------|
| CPU | 32-bit RISC-V, 320MHz | Same |
| Wi-Fi | 802.11a/b/g/n/ac/ax | Same |
| BLE | 5.0 + Mesh | Same |
| GPIO | 31 pins | 35 pins |
| UART | 2 channels | Same |
| SPI | 1 channel | Same |
| I2C | 1 channel | Same |
| PWM | 5 channels | Same |
| ADC | 12-bit SAR | Same |
| DMA | 8 channels | Same |

---

## SKILL Directory Structure

```
bl616_618/
├── SKILL.md              # Main entry (environment setup, programming guide)
└── references/           # API Reference Documents
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

## Driver Architecture

BL616/BL618 uses the **LHAL (Low Level Hardware Abstraction Layer)** driver framework:

```c
#include "bflb_gpio.h"
#include "bflb_uart.h"

// Get device handle
struct bflb_device_s *gpio = bflb_device_get_by_name("gpio");

// Initialize GPIO
bflb_gpio_init(gpio, GPIO_PIN_0, GPIO_MODE_OUTPUT_PP, GPIO_PULL_UP);

// Operate GPIO
bflb_gpio_set(gpio, GPIO_PIN_0);
```

---

## FreeRTOS Programming Essentials

Notes for FreeRTOS applications on BL616/BL618:

1. **No need to call `vTaskStartScheduler()`** — The scheduler is auto-started by the system
2. **Main loop must call `vTaskDelay`** — otherwise the system cannot schedule other tasks
3. **Always use `vTaskDelay(pdMS_TO_TICKS(ms))` for delays**

```c
void my_task(void *param)
{
    while (1) {
        // Business logic
        bflb_gpio_toggle(gpio, GPIO_PIN_0);
        vTaskDelay(pdMS_TO_TICKS(500));  // Must delay
    }
}

void app_main(void)
{
    xTaskCreate(my_task, "blink", 512, NULL, 5, NULL);
    // No need for vTaskStartScheduler()
}
```

---

## Chip Comparison

| Item | BL602 | BL616/BL618 |
|------|-------|-------------|
| Wi-Fi | 802.11b/g/n | **Wi-Fi 6** (ax) |
| BLE | 5.0 | 5.0 + **Mesh** |
| CPU Frequency | 40MHz | **320MHz** |
| GLB_BASE | 0x40000000 | **0x20000000** |
| GPIO Layout | Shared register per 2 pins | **Independent register per pin** |
| Driver Framework | HOSAL | **LHAL** |

---

## Resource Links

| Resource | Link |
|-----|------|
| Official Documentation | https://dev.bouffalolab.com/ |
| Bouffalo SDK | https://github.com/bouffalolab/bouffalo_sdk |
| BL616 Datasheet | https://dev.bouffalolab.com/media/doc/bl616/datasheet |
| BL618 Datasheet | https://dev.bouffalolab.com/media/doc/bl618/datasheet |
