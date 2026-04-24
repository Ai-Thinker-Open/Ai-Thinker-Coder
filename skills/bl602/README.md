# Ai-Thinker-Coder-bl602

Ai-WB2 series development guide based on BL602 chip.

## Overview

Ai-WB2 is a Wi-Fi & BLE dual-mode module based on **BL602 chip** (Bouffalolab), supporting:
- Wi-Fi 802.11b/g/n, 20MHz bandwidth, up to 72.2 Mbps
- Bluetooth Low Energy 5.0 + Bluetooth Mesh
- 32-bit RISC CPU (276KB RAM)
- Multiple sleep modes, deep sleep current 12μA

## Installation

This sub-skill is part of Ai-Thinker-Coder. Install the parent skill first, or install this sub-skill directly.

### Hermes Agent

```bash
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-Coder.git ~/.hermes/profiles/<YOUR_PROFILE>/skills/hardware/Ai-Thinker-Coder
```

### Trae

```bash
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-Coder.git ~/.trae/skills/Ai-Thinker-Coder
```

### CodeBuddy

```bash
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-Coder.git ~/.codebuddy/skills-marketplace/skills/Ai-Thinker-Coder
```

### OpenClaw

```bash
openclaw skill install Ai-Thinker-Open/Ai-Thinker-Coder-bl602
```

## Usage

```
/skill Ai-Thinker-Coder-bl602
```

## Module Selection

| Model | Package | Flash | RAM | Antenna | Features |
|-------|---------|-------|-----|---------|----------|
| Ai-WB2-01F | SMD | 2MB | 276KB | On-board PCB | Compatible with ESP8285 |
| Ai-WB2-01M | SMD | 2MB | 276KB | Stamp hole | - |
| Ai-WB2-01S | SMD | 2MB | 276KB | IPEX | - |
| Ai-WB2-05W | SMD | 4MB | 276KB | On-board PCB | - |
| Ai-WB2-07S | SMD | 4MB | 276KB | IPEX | - |
| Ai-WB2-12F | SMD | 4MB | 276KB | On-board PCB | Cost-effective |
| Ai-WB2-12S | SMD | 4MB | 276KB | IPEX | - |
| Ai-WB2-13 | SMD | 4MB | 276KB | Ceramic antenna | - |
| Ai-WB2-13U | SMD | 4MB | 276KB | USB | With USB port |
| Ai-WB2-32S | SMD | 4MB | 276KB | IPEX | - |

## Development Environment

### Windows

1. Install VSCode: https://code.visualstudio.com/
2. Install MSYS2: https://www.msys2.org/
3. In MSYS2 MINGW64 terminal:

```bash
pacman -S mingw-w64-x86_64-gcc mingw-w64-x86_64-gdb mingw-w64-x86_64-make
pacman -S python3 python3-pip
pip install pyserial
```

### Linux (Ubuntu 20.04) / WSL

```bash
sudo apt update
sudo apt install -y build-essential git python3 python3-pip python3-dev wget sed
```

### Get SDK

```bash
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-WB2.git
cd Ai-Thinker-WB2
git submodule update --init --recursive
```

> **China mainland users**: if GitHub is inaccessible, use the Gitee mirror:
> ```bash
> git clone https://gitee.com/aithinker_open/Ai-Thinker-WB2.git
> cd Ai-Thinker-WB2
> git submodule update --init --recursive
> ```

### Build

```bash
cd applications/get-started/helloworld
make -j8
```

### Flash

```bash
make flash p=/dev/ttyUSB0 b=921600
```

## GPIO Register Programming

**SDK Path**: `~/projects/Ai-Thinker-WB2`（示例路径，用户需替换为自己克隆的路径）

**GPIO Base Address**:
- GLB_BASE = 0x40000000
- GPIO config register: GLB_BASE + 0x100 + (pin/2)*4
- GPIO output value: GLB_BASE + 0x188
- GPIO output enable: GLB_BASE + 0x190

**Configuration Example**:

```c
#define GLB_BASE  0x40000000
#define GLB_REG(off) (*(volatile uint32_t *)(GLB_BASE + off))

static void gpio_set_output(uint8_t pin) {
    uint32_t reg_off = 0x100 + (pin/2)*4;
    uint32_t tmp = GLB_REG(reg_off);
    
    if (pin % 2 == 0) {
        tmp &= ~(1 << 0);           // Clear IE
        tmp &= ~(0xF << 8);         // Clear FUNC_SEL
        tmp |= (11 << 8);           // Set to GPIO mode
    } else {
        tmp &= ~(1 << 16);          // Clear IE
        tmp &= ~(0xF << 24);        // Clear FUNC_SEL
        tmp |= (11 << 24);          // Set to GPIO mode
    }
    GLB_REG(reg_off) = tmp;
    GLB_REG(0x190) |= (1 << pin);  // Enable output
}

static void gpio_write(uint8_t pin, uint8_t val) {
    if (val) GLB_REG(0x188) |= (1 << pin);
    else     GLB_REG(0x188) &= ~(1 << pin);
}
```

## Flash Tools

| Tool | Download | Description |
|------|----------|-------------|
| BL602 Flash Download Tool | [Download](https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/bl602_flash_download_tool.zip) | Official tool |
| Development Board Tool | [Download](https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/v1.7.4-release.zip) | With GUI |

## Links

- **Selection Table**: https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/ai-wb2_selection_table.xlsx
- **Flash Tutorial**: https://blog.csdn.net/Boantong_/article/details/125781602
