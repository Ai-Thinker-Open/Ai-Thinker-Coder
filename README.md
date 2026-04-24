# Ai-Thinker-Coder

Hermes Agent skill for Ai-Thinker IoT module development.

## Overview

This is a comprehensive skill collection for Ai-Thinker IoT modules, covering WiFi, BLE, LoRa, Radar, NB-IoT, and NearLink modules. Sub-skills are organized by chip platform.

## Platform-Specific Installation

### Hermes Agent (Recommended)

Hermes supports two install methods:

**Method 1: git clone (manual)**

```bash
# Clone to Hermes skills directory
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-Coder.git \
  ~/.hermes/profiles/<YOUR_PROFILE>/skills/hardware/Ai-Thinker-Coder
```

**Method 2: hermes tap (auto-discover)**

```bash
# Add the GitHub repo as a skill source
hermes skills tap add Ai-Thinker-Open/Ai-Thinker-Coder

# Then install from tap
hermes skills install Ai-Thinker-Open/Ai-Thinker-Coder/skills
```

**Load the skill:**

```
/skill Ai-Thinker-Coder
```

### Trae

Trae IDE stores skills at the **project level** in `.trae/skills/`:

```bash
# Project-level installation
mkdir -p <your-project>/.trae/skills
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-Coder.git \
  <your-project>/.trae/skills/Ai-Thinker-Coder
```

Trae will automatically discover skills in the `.trae/skills/` directory.

### CodeBuddy

```bash
# Clone to CodeBuddy skills marketplace
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-Coder.git \
  ~/.codebuddy/skills-marketplace/skills/Ai-Thinker-Coder
```

Skills in the marketplace are automatically indexed by CodeBuddy.

### OpenClaw (CLI)

OpenClaw uses its own command-line installation tool:

```bash
# Install via openclaw CLI
openclaw skill install Ai-Thinker-Open/Ai-Thinker-Coder

# Or specify the GitHub repository directly
openclaw skill install --github Ai-Thinker-Open/Ai-Thinker-Coder
```

## Sub-Skill Installation

Sub-skills are organized by chip platform. You can install the main skill (which includes all sub-skills) or install specific chip skills individually.

### Hermes / Trae / CodeBuddy (git clone)

```bash
# Install all sub-skills via main repo
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-Coder.git <skills-dir>/Ai-Thinker-Coder

# Paths by platform:
# Hermes:    ~/.hermes/profiles/<YOUR_PROFILE>/skills/hardware/
# Trae:      <your-project>/.trae/skills/
# CodeBuddy: ~/.codebuddy/skills-marketplace/skills/
```

### OpenClaw (CLI)

```bash
# Install main skill (includes all sub-skills)
openclaw skill install Ai-Thinker-Open/Ai-Thinker-Coder

# Install specific sub-skill
openclaw skill install Ai-Thinker-Open/Ai-Thinker-Coder-bl602
```

## Usage

### Load Main Skill

```
/skill Ai-Thinker-Coder
```

Displays product overview and links to sub-skills.

### Load Chip-Specific Skill

```
/skill Ai-Thinker-Coder-bl602    # Ai-WB2 series (BL602 chip)
```

### Available Sub-Skills

| Skill | Chip Platform | Product Series |
|-------|-------------|----------------|
| Ai-Thinker-Coder-bl602 | BL602 | Ai-WB2-01S/12F/32S |
| Ai-Thinker-Coder-bl618 | BL616/BL618 | Ai-M61/M62 series |
| Ai-Thinker-Coder-lora | - | Ra-01/RA-01H LoRa |
| Ai-Thinker-Coder-radar | - | RD-01/03/04 Radar |

## Development Workflow

1. **Identify your module** - Check the chip platform on your Ai-Thinker module
2. **Load corresponding skill** - Use `/skill Ai-Thinker-Coder-<chip>`
3. **Follow setup guide** - Configure development environment
4. **Build and flash** - Use provided Makefile and flash commands

## Quick Start Example (BL602/Ai-WB2)

```bash
# 1. Load the skill
/skill Ai-Thinker-Coder-bl602

# 2. Clone SDK
cd ~/projects
git clone https://github.com/Ai-Thinker-Open/Ai-Thinker-WB2.git
cd Ai-Thinker-WB2
git submodule update --init --recursive

# 3. Build example
cd applications/get-started/helloworld
make

# 4. Flash firmware
make flash p=/dev/ttyUSB0 b=921600
```

## Documentation Structure

```
Ai-Thinker-Coder/
├── SKILL.md              # Main entry (Chinese)
├── README.md             # English installation guide
├── README_Zh.md         # Chinese installation guide
└── skills/
    └── bl602/
        ├── SKILL.md      # BL602 detailed guide
        ├── README.md     # English usage guide
        └── README_Zh.md  # Chinese usage guide
```

## Requirements

- Git for version control
- USB serial connection to module for hardware development
- For WSL development: usbipd configured for device passthrough
- Platform-specific CLI (openclaw for OpenClaw platform)

## Troubleshooting

- **Skill not found**: Ensure skill is in the correct platform-specific skills directory
- **Build errors**: Check git submodules are initialized (`git submodule update --init --recursive`)
- **Flash failures**: Verify BOOT pin pulled low and serial port correct

## Links

- **Ai-Thinker Website**: https://www.ai-thinker.com
- **Forum**: https://bbs.ai-thinker.com
- **Docs**: https://docs.ai-thinker.com
- **GitHub**: https://github.com/Ai-Thinker-Open/

## License

MIT License - See individual sub-skill licenses for details.
