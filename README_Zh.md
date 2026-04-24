# Ai-Thinker-Coder

Hermes Agent 安信可科技物联网模组开发助手。

## 概述

本 skill 集合涵盖安信可全系列模组开发指南，包括 WiFi、BLE、LoRa、Radar、NB-IoT、星闪等。子 skill 按芯片平台分组，便于针对性开发。

## 平台化安装方法

### Hermes Agent（推荐）

Hermes 支持两种安装方式：

**方式一：git clone（手动）**

```bash
# 克隆到 Hermes skills 目录
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-Coder.git \
  ~/.hermes/profiles/<YOUR_PROFILE>/skills/hardware/Ai-Thinker-Coder
```

**方式二：hermes tap（自动发现）**

```bash
# 添加 GitHub 仓库为 skill 数据源
hermes skills tap add Ai-Thinker-Open/Ai-Thinker-Coder

# 从 tap 安装
hermes skills install Ai-Thinker-Open/Ai-Thinker-Coder/skills
```

**加载 skill：**

```
/skill Ai-Thinker-Coder
```

### Trae

Trae IDE 将 skills 存储在**项目级别**的 `.trae/skills/` 目录：

```bash
# 项目级别安装
mkdir -p <your-project>/.trae/skills
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-Coder.git \
  <your-project>/.trae/skills/Ai-Thinker-Coder
```

Trae 会自动发现 `.trae/skills/` 目录下的 skills。

### CodeBuddy

```bash
# 克隆到 CodeBuddy skills marketplace
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-Coder.git \
  ~/.codebuddy/skills-marketplace/skills/Ai-Thinker-Coder
```

marketplace 中的 skills 会被 CodeBuddy 自动索引。

### OpenClaw（命令行）

OpenClaw 使用独立的命令行工具安装 skills：

```bash
# 通过 openclaw CLI 安装
openclaw skill install Ai-Thinker-Open/Ai-Thinker-Coder

# 或直接指定 GitHub 仓库
openclaw skill install --github Ai-Thinker-Open/Ai-Thinker-Coder
```

## 子 Skill 安装

子 skill 按芯片平台组织。你可以安装主 skill（包含所有子 skill）或单独安装特定芯片的 skill。

### Hermes / Trae / CodeBuddy（git clone）

```bash
# 通过主仓库安装所有子 skill
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-Coder.git <skills-dir>/Ai-Thinker-Coder

# 各平台路径：
# Hermes:    ~/.hermes/profiles/<YOUR_PROFILE>/skills/hardware/
# Trae:      <your-project>/.trae/skills/
# CodeBuddy: ~/.codebuddy/skills-marketplace/skills/
```

### OpenClaw（命令行）

```bash
# 安装主 skill（包含所有子 skill）
openclaw skill install Ai-Thinker-Open/Ai-Thinker-Coder

# 安装特定子 skill
openclaw skill install Ai-Thinker-Open/Ai-Thinker-Coder-bl602
```

## 使用方法

### 加载主 skill

```
/skill Ai-Thinker-Coder
```

显示产品总览和子 skill 链接。

### 加载芯片专属 skill

```
/skill Ai-Thinker-Coder-bl602    # Ai-WB2 系列 (BL602 芯片)
```

### 可用子 skill

| Skill | 芯片平台 | 产品系列 |
|-------|---------|---------|
| Ai-Thinker-Coder-bl602 | BL602 | Ai-WB2-01S/12F/32S |
| Ai-Thinker-Coder-bl618 | BL616/BL618 | Ai-M61/M62 系列 |
| Ai-Thinker-Coder-esp32 | ESP32/ESP8266 | ESP-12F/ESP32-S3 |
| Ai-Thinker-Coder-lora | - | Ra-01/RA-01H LoRa |
| Ai-Thinker-Coder-radar | - | RD-01/03/04 Radar |

## 开发流程

1. **确认模组型号** - 查看模组上的芯片平台标识
2. **加载对应 skill** - 使用 `/skill Ai-Thinker-Coder-<芯片型号>`
3. **按照指南搭建环境** - 配置开发工具链
4. **编译并烧录** - 使用提供的 Makefile 和烧录命令

## 快速开始示例 (BL602/Ai-WB2)

```bash
# 1. 加载 skill
/skill Ai-Thinker-Coder-bl602

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
└── skills/
    └── bl602/
        ├── SKILL.md      # BL602 详细开发指南
        ├── README.md     # 英文使用指南
        └── README_Zh.md  # 中文使用指南
```

## 前置要求

- Git 版本控制工具
- 硬件开发：USB 串口连接模组
- WSL 开发：配置 usbipd 进行设备穿透
- OpenClaw 平台需安装 openclaw CLI

## 常见问题

- **Skill 找不到**：确认 skill 位于对应平台的 skills 目录下
- **编译报错**：检查 git 子模块是否已初始化 (`git submodule update --init --recursive`)
- **烧录失败**：确认 BOOT 引脚拉低，串口选择正确

## 相关链接

- **安信可官网**: https://www.ai-thinker.com
- **技术支持论坛**: https://bbs.ai-thinker.com
- **文档中心**: https://docs.ai-thinker.com
- **GitHub**: https://github.com/Ai-Thinker-Open/

## 许可证

MIT 许可证 - 详见各子 skill 的许可证说明。
