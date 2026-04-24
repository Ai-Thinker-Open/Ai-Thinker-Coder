# Ai-Thinker-Coder-bl602

安信可 Ai-WB2 系列开发指南 (BL602 芯片平台)。

## 概述

Ai-WB2 系列是安信可科技开发的 Wi-Fi & BLE 双模模组，基于 **BL602 芯片** (Bouffalolab)，支持：
- Wi-Fi 802.11b/g/n，20MHz 带宽，最高 72.2 Mbps
- Bluetooth Low Energy 5.0 + Bluetooth Mesh
- 32-bit RISC CPU (276KB RAM)
- 多种休眠模式，深度睡眠电流 12μA

## 安装

本子 skill 是 Ai-Thinker-Coder 的一部分。请先安装主 skill 或直接安装此子 skill。

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

## 使用方法

```
/skill Ai-Thinker-Coder-bl602
```

## 模组选型

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

## 开发环境

### Windows

1. 安装 VSCode：https://code.visualstudio.com/
2. 安装 MSYS2：https://www.msys2.org/
3. 在 MSYS2 MINGW64 终端中执行：

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

### 获取 SDK

```bash
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-WB2.git
cd Ai-Thinker-WB2
git submodule update --init --recursive
```

> **国内用户**：如果 GitHub 访问不稳定，可使用 Gitee 镜像：
> ```bash
> git clone https://gitee.com/aithinker_open/Ai-Thinker-WB2.git
> cd Ai-Thinker-WB2
> git submodule update --init --recursive
> ```

### 设置工具链权限

```bash
cd toolchain/riscv/Linux/
bash chmod755.sh
```

### 编译

```bash
cd applications/get-started/helloworld
make -j8
```

### 烧录

```bash
make flash p=/dev/ttyUSB0 b=921600
```

## GPIO 寄存器级编程

**SDK 路径**：`~/projects/Ai-Thinker-WB2`（示例路径，用户需替换为自己克隆的路径）

**GPIO 基础地址**：
- GLB_BASE = 0x40000000
- GPIO 配置寄存器：GLB_BASE + 0x100 + (pin/2)*4
- GPIO 输出值：GLB_BASE + 0x188
- GPIO 输出使能：GLB_BASE + 0x190

**配置示例**：

```c
#define GLB_BASE  0x40000000
#define GLB_REG(off) (*(volatile uint32_t *)(GLB_BASE + off))

static void gpio_set_output(uint8_t pin) {
    uint32_t reg_off = 0x100 + (pin/2)*4;
    uint32_t tmp = GLB_REG(reg_off);
    
    if (pin % 2 == 0) {
        tmp &= ~(1 << 0);           // 清除 IE
        tmp &= ~(0xF << 8);         // 清除 FUNC_SEL
        tmp |= (11 << 8);           // 设置为 GPIO 模式
    } else {
        tmp &= ~(1 << 16);          // 清除 IE
        tmp &= ~(0xF << 24);        // 清除 FUNC_SEL
        tmp |= (11 << 24);          // 设置为 GPIO 模式
    }
    GLB_REG(reg_off) = tmp;
    GLB_REG(0x190) |= (1 << pin);  // 使能输出
}

static void gpio_write(uint8_t pin, uint8_t val) {
    if (val) GLB_REG(0x188) |= (1 << pin);
    else     GLB_REG(0x188) &= ~(1 << pin);
}
```

## 烧录工具

| 工具 | 下载 | 说明 |
|-----|------|-----|
| BL602 Flash Download Tool | [点击下载](https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/bl602_flash_download_tool.zip) | 官方工具 |
| 开发板专用工具 | [点击下载](https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/v1.7.4-release.zip) | 带 GUI |

**详细教程**：https://blog.csdn.net/Boantong_/article/details/125781602

## 相关链接

- **选型表下载**：https://aithinker-static.oss-cn-shenzhen.aliyuncs.com/docs/_media_old/ai-wb2_selection_table.xlsx
