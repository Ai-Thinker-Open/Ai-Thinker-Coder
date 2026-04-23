# Ai-Thinker-Coder

Hermes Agent 安信可科技物联网模组开发助手。

## 概述

本 skill 集合涵盖安信可全系列模组开发指南，包括 WiFi、BLE、LoRa、Radar、NB-IoT、星闪等。子 skill 按芯片平台分组，便于针对性开发。

## 安装方法

### 方法一：通过 Hermes Agent 安装（推荐）

```
/skill install Ai-Thinker-Coder
```

### 方法二：手动安装

```bash
# 克隆 skill 仓库到 Hermes skills 目录
git clone <仓库地址> ~/.hermes/profiles/<YOUR_PROFILE>/skills/hardware/Ai-Thinker-Coder

# 或手动复制
cp -r Ai-Thinker-Coder ~/.hermes/profiles/<YOUR_PROFILE>/skills/hardware/
```

### 方法三：单独安装子 skill

```bash
# 安装特定芯片的 skill
/skill install Ai-Thinker-Coder/bl602    # Ai-WB2 (BL602)

# 或手动复制
cp -r Ai-Thinker-Coder/bl602 ~/.hermes/profiles/<YOUR_PROFILE>/skills/hardware/
```

## 使用方法

### 加载主 skill

```
/kill Ai-Thinker-Coder
```

显示产品总览和子 skill 链接。

### 加载芯片专属 skill

```
/skill Ai-Thinker-Coder/bl602    # Ai-WB2 系列 (BL602 芯片)
```

### 可用子 skill

| Skill | 芯片平台 | 产品系列 |
|-------|---------|---------|
| Ai-Thinker-Coder/bl602 | BL602 | Ai-WB2-01S/12F/32S |
| Ai-Thinker-Coder/bl618 | BL616/BL618 | Ai-M61/M62 系列 |
| Ai-Thinker-Coder/esp32 | ESP32/ESP8266 | ESP-12F/ESP32-S3 |
| Ai-Thinker-Coder/lora | - | Ra-01/RA-01H LoRa |
| Ai-Thinker-Coder/radar | - | RD-01/03/04 Radar |

## 开发流程

1. **确认模组型号** - 查看模组上的芯片平台标识
2. **加载对应 skill** - 使用 `/skill Ai-Thinker-Coder-<芯片型号>`
3. **按照指南搭建环境** - 配置开发工具链
4. **编译并烧录** - 使用提供的 Makefile 和烧录命令

## 快速开始示例 (BL602/Ai-WB2)

```bash
# 1. 加载 skill
/skill Ai-Thinker-Coder/bl602

# 2. 克隆 SDK
cd /home/seahi/workspase
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-WB2.git
cd Ai-Thinker-WB2
git submodule update --init --recursive

# 3. 编译示例
cd applications/get-started/helloworld
make

# 4. 烧录固件
make flash p=/dev/ttyUSB0 b=921600
```

## 目录结构

```
Ai-Thinker-Coder/
├── SKILL.md              # 主入口（中文）
├── README.md             # 英文安装指南
├── README_Zh.md         # 中文安装指南
└── Ai-Thinker-Coder/bl602/
    └── SKILL.md          # BL602 详细开发指南
```

## 前置要求

- 已启用 skill 管理的 Hermes Agent
- 硬件开发：USB 串口连接模组
- WSL 开发：配置 usbipd 进行设备穿透

## 常见问题

- **Skill 找不到**：确认 skill 位于 `~/.hermes/profiles/<YOUR_PROFILE>/skills/hardware/` 目录
- **编译报错**：检查 git 子模块是否已初始化 (`git submodule update --init --recursive`)
- **烧录失败**：确认 BOOT 引脚拉低，串口选择正确

## 相关链接

- **安信可官网**: https://www.ai-thinker.com
- **技术支持论坛**: https://bbs.ai-thinker.com
- **文档中心**: https://docs.ai-thinker.com
- **GitHub**: https://github.com/Ai-Thinker-Open/

## 许可证

MIT 许可证 - 详见各子 skill 的许可证说明。
