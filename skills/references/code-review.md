# BL602 代码审查规范

> 适用：BL602 (Ai-WB2) 模组项目，编译成功后必须完成代码审查
> 依据：安信可 C 语言编码规范（axk 函数头标准）
> 范围：代码格式、axk 函数头规范性、业务逻辑、注释质量

---

## 一、审查时机与流程

```
编译成功 ──> 代码审查 ──> 审查通过 ──> 代码提交/发布
                 │
                 └── 审查不通过 ──> 修复 ──> 重新编译 ──> 再次审查
```

**审查触发条件**：每次 `make` 或 `bflb_mkimg` 编译成功后，必须自动进入代码审查阶段。

---

## 二、审查前置准备

### 2.1 获取变更文件列表

```bash
# 仅审查本次修改的文件（推荐）
git diff --name-only

# 审查自上一个可靠版本以来的所有变更
git diff HEAD~1 --name-only

# 审查自 tag/v1.0.0 以来的所有变更
git diff v1.0.0 --name-only
```

### 2.2 获取完整 Diff

```bash
# 获取变更统计（先看范围）
git diff --stat

# 获取完整变更内容
git diff
```

### 2.3 设置项目关键路径（按需调整）

```bash
PROJECT_ROOT="/path/to/project"
APP_DIR="$PROJECT_ROOT/app"
SDK_COMPONENTS="$PROJECT_ROOT/components"
```

---

## 三、格式审查（依据 axk 编码规范）

### 3.1 axk 函数头规范性审查

审查每个 C 文件是否满足以下要求：

| 检查项 | 检查命令 | 通过标准 |
|--------|---------|---------|
| 所有函数都有 axk 函数头 | `grep -rn "^[^/].*{" --include="*.c" \| grep -v "^/\*" \| grep -v "^//"` | 每个函数前50行内有 `axk_` 函数头 |
| 函数头包含必需字段 | 检查 `@brief`/`@param`/`@return`/`@author`/`@date` | 五字段完整 |
| 函数名与函数头首行前缀一致 | `grep "axk_"` | 函数名 `axk_xxx` 与函数头首行 `axk_xxx：`一致 |
| 静态函数注明"静态函数" | `grep -n "static.*axk_\|^/\*.*静态函数"` | 静态函数 @note 包含"静态函数，仅本文件内可调用" |

**快速检查脚本**：

```bash
# 检查函数头完整性（简单版）
for f in $(find . -name "*.c" -not -path "./components/*"); do
    echo "=== Checking $f ==="
    # 检查是否有函数缺少函数头
    grep -n "^[a-zA-Z].*({" "$f" | while read line; do
        func_line=$(echo "$line" | cut -d: -f1)
        prev_lines=$(sed -n "$((func_line > 20 ? func_line - 20 : 1)),$((func_line-1))p" "$f")
        if ! echo "$prev_lines" | grep -q "axk_"; then
            echo "  WARNING: Function at line $func_line may lack axk header"
        fi
    done
done
```

### 3.2 命名规范审查

| 类型 | 正确 | 错误 |
|------|------|------|
| 函数名 | `axk_uart_init`, `axk_gpio_set_level` | `UartInit`, `gpio_set` |
| 全局变量 | `g_axk_system_status` | `systemStatus`, `global_cnt` |
| 静态变量 | `s_axk_temp_buf` | `tempBuffer`, `static_val` |
| 宏定义 | `AXK_UART_BAUD_115200` | `uart_baud`, `BAUD_RATE` |
| 枚举/结构体 | `AxkStatus`, `AXK_STATUS_OK` | `status_t`, `STATUS_OK` |

**检查命令**：

```bash
# 检查函数命名（不以 axk_ 开头）
grep -rn "^[a-zA-Z_].*(.*)" --include="*.c" | \
    grep -v "^.*axk_" | \
    grep -v "^.*//" | \
    grep -v "^.*/\*" | \
    grep -v "struct\|typedef\|enum"

# 检查全局变量（不以 g_axk_ 开头）
grep -rn "^[a-zA-Z].*\s" --include="*.c" | \
    grep -v "^.*axk_" | \
    grep -v "static\|const\|//\|/\*"
```

### 3.3 代码格式审查

| 检查项 | 通过标准 | 检查方法 |
|--------|---------|---------|
| 缩进 | 4 空格，非 Tab | `grep -rn $'\t' --include="*.c"` 应无结果 |
| if/for/while 单行也加 `{}` | `if (x) func();` 为错误 | `grep -rn "if.*)[^}]*;$" --include="*.c"` |
| 运算符周围有空格 | `a+b` 错误，`a + b` 正确 | `grep -rn "=[^= \t]" --include="*.c"` |
| 行长度 | 不超过 120 字符（建议） | `awk 'length>120 {print NR": "$0}' *.c` |

---

## 四、业务逻辑审查

### 4.1 通用嵌入式逻辑检查

| 检查项 | 风险 | 正确做法 |
|--------|------|---------|
| 延时函数 | 死机/任务无法调度 | `vTaskDelay(pdMS_TO_TICKS(ms))` |
| 空循环无延时 | CPU 占用 100%，系统假死 | 主循环必须有 `vTaskDelay` |
| 指针使用前校验 | 野指针崩溃 | `if (ptr == NULL) return;` |
| 数组越界访问 | 内存破坏 | 检查边界条件 |
| 外设未初始化先使用 | 硬件异常 | 初始化顺序正确 |

**关键危险模式检测命令**：

```bash
# 检测无延时的 while 循环（可能导致系统假死）
grep -rn "while.*)" --include="*.c" -A2 | grep -v "vTaskDelay\|msleep\|delay\|sleep\|for\|if"

# 检测空 for 循环无延时
grep -rn "for.*;" --include="*.c" -A1 | grep -v "vTaskDelay\|msleep\|delay"

# 检测可疑的 hardfault 风险模式
grep -rn "NULL\)" --include="*.c" | grep -v "if.*NULL\|== NULL\|!= NULL"

# 检测未检查返回值的函数调用
grep -rn "axk_.*(" --include="*.c" | grep -v "if.*axk_\|=\s*axk_\|\s*axk_.*;"

# 检测 delay 函数的正确性（应为 vTaskDelay）
grep -rn "vTaskDelay\|pvPortDelay" --include="*.c"
```

### 4.2 FreeRTOS 专项检查

| 检查项 | 风险 | 正确做法 |
|--------|------|---------|
| 主循环无 `vTaskDelay` | 系统假死，无法调度其他任务 | 主循环必须有 `vTaskDelay` |
| 在中断中使用 FreeRTOS API | 行为未定义 | 中断中仅用 `FromISR` 版本函数 |
| 栈空间估算 | 栈溢出 | 合理估算任务栈用量 |
| 临界区嵌套 | 系统死锁 | `taskENTER_CRITICAL()` 成对出现 |

**FreeRTOS 专项检查命令**：

```bash
# 检测主循环是否调用了 vTaskDelay
grep -rn "vTaskDelay" --include="*.c"

# 检测是否有中断中使用 FreeRTOS API（不安全）
grep -rn "xQueueSend\|xSemaphoreGive\|vTaskDelay" --include="*.c" | \
    grep -i "interrupt\|isr\|irq"

# 检测临界区是否配对
grep -rn "taskENTER_CRITICAL\|taskEXIT_CRITICAL" --include="*.c"
```

### 4.3 外设驱动审查

| 检查项 | 风险 | 正确做法 |
|--------|------|---------|
| GPIO 操作前已初始化 | 行为不确定 | `hosal_gpio_init()` 成功后才能操作 |
| UART 收发buffersize合理 | 内存浪费/溢出 | buffersize 与实际需求匹配 |
| I2C 从机地址正确 | 通信失败 | 地址检查 |
| SPI 片选时序 | 数据错位 | 片选先低后高完整 |

---

## 五、注释质量审查

### 5.1 必须有注释的关键位置

| 位置 | 要求 |
|------|------|
| 所有 `axk_` 函数头 | 完整五字段 |
| 复杂业务逻辑 | `// 为什么这样做` 而非 `// 做什么` |
| 条件分支边界 | `if (x > 100) // x 超过阈值则进入异常处理` |
| 寄存器操作 | `// GLB->GPIO_CFG[0] |= (1 << 8); // 使能 GPIO0 输出` |
| 硬件相关常量 | `// XTAL频率: 40000000 Hz` |

### 5.2 禁止的注释类型

```bash
# 无意义注释（禁止）
grep -rn "// 循环\|// 赋值\|// 加一\|// 初始化" --include="*.c"

// TODO/FIXME/HACK 未注明日期和作者（建议补全）
grep -rn "TODO\|FIXME\|HACK" --include="*.c"

// 被注释掉的废弃代码（建议直接删除）
grep -rn "^//.*axk_\|^/\*.*axk_" --include="*.c"
```

---

## 六、审查输出格式

每次审查完成后，按以下格式输出结果：

```
## BL602 代码审查报告

审查范围: app/ 目录，commit: abc1234
审查时间: 2026-04-27 10:30
审查依据: 安信可 C 语言编码规范 (axk)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔴 Critical（必须修复）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[文件:行号] 问题描述
  → 建议修复方式

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️ Warning（建议修复）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[文件:行号] 问题描述
  → 建议修复方式

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💡 Suggestion（可选优化）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[文件:行号] 问题描述
  → 优化建议

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Passed（通过检查）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- axk 函数头完整性: PASS
- 命名规范: PASS
- 空格/Tab缩进: PASS
- FreeRTOS vTaskDelay: PASS

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
审查结论: [通过 / 需要修复后重审]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 七、自动审查脚本（参考实现）

```bash
#!/bin/bash
# bl602_code_review.sh - BL602 编译后自动代码审查脚本
# 用法: ./bl602_code_review.sh [app_dir]

set -e

APP_DIR="${1:-app}"
REPORT="/tmp/bl602_review_$(date +%Y%m%d_%H%M%S).txt"

echo "BL602 代码审查开始..."

# 1. axk 函数头检查
echo "=== 1. axk 函数头检查 ===" >> "$REPORT"
find "$APP_DIR" -name "*.c" | while read f; do
    func_count=$(grep -c "^[^/].*({" "$f" 2>/dev/null || echo 0)
    header_count=$(grep -c "axk_" "$f" 2>/dev/null || echo 0)
    if [ "$header_count" -lt "$func_count" ]; then
        echo "  [WARN] $f: $func_count 个函数, 仅 $header_count 处 axk 引用" >> "$REPORT"
    fi
done

# 2. Tab 缩进检查
echo "=== 2. Tab 缩进检查 ===" >> "$REPORT"
tab_count=$(grep -rn $'\t' "$APP_DIR" --include="*.c" 2>/dev/null | tee /dev/stderr | wc -l)
if [ "$tab_count" -gt 0 ]; then
    echo "  [FAIL] 发现 $tab_count 处 Tab 缩进" >> "$REPORT"
else
    echo "  [PASS] 无 Tab 缩进" >> "$REPORT"
fi

# 3. vTaskDelay 检查
echo "=== 3. FreeRTOS vTaskDelay 检查 ===" >> "$REPORT"
delay_count=$(grep -rn "vTaskDelay" "$APP_DIR" --include="*.c" 2>/dev/null | wc -l)
echo "  找到 $delay_count 处 vTaskDelay 调用" >> "$REPORT"

# 4. 无延时循环检查
echo "=== 4. 危险循环检查 ===" >> "$REPORT"
grep -rn "while.*)" "$APP_DIR" --include="*.c" -A2 2>/dev/null | \
    grep -v "vTaskDelay\|for\|if\|break\|return" >> "$REPORT" || true

echo ""
echo "审查报告: $REPORT"
cat "$REPORT"
```

---

## 八、审查核对清单

审查完成后，逐项核对：

### 格式规范
- [ ] 所有函数有完整 axk 函数头（`@brief`/`@param`/`@return`/`@author`/`@date`）
- [ ] 函数名以 `axk_` 开头，与函数头首行一致
- [ ] 静态函数 @note 注明"静态函数，仅本文件内可调用"
- [ ] 全局变量以 `g_axk_` 开头，静态变量以 `s_axk_` 开头
- [ ] 宏定义以 `AXK_` 开头，全大写
- [ ] 4 空格缩进，无 Tab 字符
- [ ] if/for/while 单行也加 `{}`
- [ ] 运算符周围有空格

### 业务逻辑
- [ ] 主循环中有 `vTaskDelay`，无死循环
- [ ] 指针使用前有 NULL 检查
- [ ] 外设初始化顺序正确（GPIO → 外设）
- [ ] 延时使用 `vTaskDelay(pdMS_TO_TICKS(ms))`
- [ ] 中断中不使用普通 FreeRTOS API

### 注释质量
- [ ] 无无意义注释（如 `// 循环`）
- [ ] 无被注释掉的废弃代码
- [ ] TODO/FIXME 有日期和作者
- [ ] 寄存器操作有注释说明

### 安全性
- [ ] 参数有合法性校验
- [ ] 数组访问有边界检查
- [ ] 无硬编码敏感信息
