# SEC_DSA API Reference (BL616/BL618)

> **Source:** `bouffalo_sdk/drivers/lhal/include/bflb_sec_dsa.h`  
> **Implementation:** `bouffalo_sdk/drivers/lhal/src/pka/libpka_bl616.a` (预编译 PKA 库)  
> **Hardware:** SEC_ENG PKA (Public Key Accelerator) @ `0x20004300`  
> **Register Header:** `bouffalo_sdk/drivers/lhal/include/hardware/sec_eng_reg.h`

## Overview

SEC_DSA 模块提供基于 **DSA (Digital Signature Algorithm)** 的数字签名和验证功能。该模块利用 SEC_ENG（安全引擎）中的 **PKA (Public Key Accelerator)** 硬件加速器执行大数模幂运算，支持 CRT (Chinese Remainder Theorem) 优化的 RSA/DSA 私钥操作。

**主要特性：**
- 基于硬件 PKA 的大数模幂运算加速
- 支持 DSA 签名生成 (`bflb_sec_dsa_sign`)
- 支持 DSA 签名验证 (`bflb_sec_dsa_verify`)
- 支持 CRT 模式以加速私钥操作 (`bflb_dsa_crt_s`)
- 支持可配置的密钥长度 (`size` / `crtSize`)
- 底层通过 `libpka_bl616.a` 预编译库实现

## Base Address

| Peripheral | Offset | Address |
|------------|--------|---------|
| SEC_ENG_BASE | — | `0x20004000` |
| SEC_ENG PKA | `0x300` | `0x20004300` |

---

## Configuration Structures

### bflb_dsa_s

DSA 密钥与操作句柄。

```c
struct bflb_dsa_s {
    uint32_t size;               // 密钥大小（32-bit words 数）
    uint32_t crtSize;            // CRT 参数大小（32-bit words 数）
    uint32_t *n;                 // 模数 n 指针
    uint32_t *e;                 // 公钥指数 e 指针
    uint32_t *d;                 // 私钥指数 d 指针
    struct bflb_dsa_crt_s crtCfg; // CRT 配置
};
```

| Field | Type | Description |
|-------|------|-------------|
| `size` | `uint32_t` | 密钥大小（以 32-bit word 为单位） |
| `crtSize` | `uint32_t` | CRT 参数大小（以 32-bit word 为单位） |
| `n` | `uint32_t *` | 模数 n（公钥和私钥共用） |
| `e` | `uint32_t *` | 公钥指数 e |
| `d` | `uint32_t *` | 私钥指数 d |
| `crtCfg` | `struct bflb_dsa_crt_s` | CRT 加速参数配置 |

### bflb_dsa_crt_s

CRT (Chinese Remainder Theorem) 参数结构，用于加速私钥操作。

```c
struct bflb_dsa_crt_s {
    uint32_t *dP;        // CRT 指数 d mod (p-1)
    uint32_t *dQ;        // CRT 指数 d mod (q-1)
    uint32_t *qInv;      // q 的模逆元 q^(-1) mod p
    uint32_t *p;         // 素数 p
    uint32_t *invR_p;    // Montgomery R 的逆元 mod p
    uint32_t *primeN_p;  // p 的素数 N (Montgomery 参数)
    uint32_t *q;         // 素数 q
    uint32_t *invR_q;    // Montgomery R 的逆元 mod q
    uint32_t *primeN_q;  // q 的素数 N (Montgomery 参数)
};
```

---

## LHAL API Functions

### bflb_sec_dsa_init

初始化 DSA 句柄，分配硬件 PKA 资源并配置密钥参数。

```c
int bflb_sec_dsa_init(struct bflb_dsa_s *handle, uint32_t size);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `struct bflb_dsa_s *` | DSA 句柄指针（含密钥参数） |
| `size` | `uint32_t` | 密钥长度（以 32-bit word 为单位） |

**Returns:** `0` 成功，负值表示错误

**说明:** 调用前需填充 `handle->n`, `handle->e`, `handle->d` 及 `handle->crtCfg` 等密钥参数。

---

### bflb_sec_dsa_sign

使用 DSA 私钥对消息哈希进行签名。

```c
int bflb_sec_dsa_sign(struct bflb_dsa_s *handle, const uint32_t *hash,
                       uint32_t hashLenInWord, uint32_t *s);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `struct bflb_dsa_s *` | 已初始化的 DSA 句柄 |
| `hash` | `const uint32_t *` | 消息哈希值（32-bit word 数组） |
| `hashLenInWord` | `uint32_t` | 哈希长度（以 32-bit word 为单位） |
| `s` | `uint32_t *` | 输出签名缓冲区 |

**Returns:** `0` 成功，负值表示错误

---

### bflb_sec_dsa_verify

使用 DSA 公钥验证签名。

```c
int bflb_sec_dsa_verify(struct bflb_dsa_s *handle, const uint32_t *hash,
                         uint32_t hashLenInWord, const uint32_t *s);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `struct bflb_dsa_s *` | 已初始化的 DSA 句柄 |
| `hash` | `const uint32_t *` | 消息哈希值（32-bit word 数组） |
| `hashLenInWord` | `uint32_t` | 哈希长度（以 32-bit word 为单位） |
| `s` | `const uint32_t *` | 待验证的签名 |

**Returns:** `0` 验证通过，负值表示验证失败或错误

---

## Usage Examples

### Example 1: DSA 签名与验证 (1024-bit)

```c
#include "bflb_sec_dsa.h"
#include "bflb_sha.h"  // 用于生成哈希

void dsa_sign_verify_1024_example(void)
{
    struct bflb_dsa_s dsa_handle;
    int ret;

    // 1024-bit 密钥 = 32 个 32-bit word
    dsa_handle.size = 32;
    dsa_handle.crtSize = 16;  // CRT 参数为 512-bit each

    // 注意: 实际应用中密钥应从安全存储加载
    // 此处仅为示例占位
    uint32_t n[32]  = { /* 模数 n */ };
    uint32_t e[32]  = { /* 公钥指数 e */ };
    uint32_t d[32]  = { /* 私钥指数 d */ };

    dsa_handle.n = n;
    dsa_handle.e = e;
    dsa_handle.d = d;

    // CRT 参数（可选，加速私钥操作）
    uint32_t p[16]     = { /* ... */ };
    uint32_t q[16]     = { /* ... */ };
    uint32_t dP[16]    = { /* ... */ };
    uint32_t dQ[16]    = { /* ... */ };
    uint32_t qInv[16]  = { /* ... */ };
    uint32_t invR_p[16] = { /* ... */ };
    uint32_t invR_q[16] = { /* ... */ };
    uint32_t primeN_p[16] = { /* ... */ };
    uint32_t primeN_q[16] = { /* ... */ };

    dsa_handle.crtCfg.p        = p;
    dsa_handle.crtCfg.q        = q;
    dsa_handle.crtCfg.dP       = dP;
    dsa_handle.crtCfg.dQ       = dQ;
    dsa_handle.crtCfg.qInv     = qInv;
    dsa_handle.crtCfg.invR_p   = invR_p;
    dsa_handle.crtCfg.invR_q   = invR_q;
    dsa_handle.crtCfg.primeN_p = primeN_p;
    dsa_handle.crtCfg.primeN_q = primeN_q;

    // 初始化 DSA
    ret = bflb_sec_dsa_init(&dsa_handle, 32);
    if (ret != 0) {
        printf("DSA init failed: %d\n", ret);
        return;
    }

    // 准备消息哈希
    uint8_t message[] = "Hello, DSA!";
    uint32_t hash[32] = {0};  // SHA-256 输出
    // 实际应用中: bflb_sha256(hash, message, sizeof(message));

    // 签名
    uint32_t signature[32] = {0};
    ret = bflb_sec_dsa_sign(&dsa_handle, hash, 8, signature);  // SHA-256 = 8 words
    if (ret == 0) {
        printf("DSA sign success\n");
    }

    // 验证
    ret = bflb_sec_dsa_verify(&dsa_handle, hash, 8, signature);
    if (ret == 0) {
        printf("DSA verify PASSED\n");
    } else {
        printf("DSA verify FAILED: %d\n", ret);
    }
}
```

### Example 2: 仅验证模式（使用公钥）

```c
#include "bflb_sec_dsa.h"

int dsa_verify_only_example(const uint32_t *hash, const uint32_t *signature)
{
    struct bflb_dsa_s dsa_handle;

    // 仅需公钥验证，私钥可置空
    uint32_t n[32] = { /* 公钥模数 */ };
    uint32_t e[32] = { /* 公钥指数 */ };

    dsa_handle.size = 32;
    dsa_handle.n = n;
    dsa_handle.e = e;
    dsa_handle.d = NULL;  // 仅验证不需要私钥

    int ret = bflb_sec_dsa_init(&dsa_handle, 32);
    if (ret != 0) return ret;

    return bflb_sec_dsa_verify(&dsa_handle, hash, 8, signature);
}
```

---

## Register-Level Reference

DSA 操作通过 SEC_ENG PKA 子系统完成。PKA 寄存器位于 `SEC_ENG_BASE + 0x300`。

### PKA Register Offsets

| Offset | Register | Description |
|--------|----------|-------------|
| `0x300` | `SE_PKA_0_CTRL_0` | PKA 控制寄存器 0 |
| `0x30C` | `SE_PKA_0_SEED` | PKA 随机种子 |
| `0x310` | `SE_PKA_0_CTRL_1` | PKA 控制寄存器 1 |
| `0x340` | `SE_PKA_0_RW` | PKA 数据读写端口 |
| `0x360` | `SE_PKA_0_RW_BURST` | PKA 批量数据读写端口 |
| `0x3FC` | `SE_PKA_0_CTRL_PROT` | PKA 访问保护 |

### PKA Control Register 0 (0x300)

| Bit(s) | Field | Description |
|--------|-------|-------------|
| 0 | `SE_PKA_0_DONE` | 操作完成标志 |
| 1 | `SE_PKA_0_DONE_CLR_1T` | 写 1 清除完成标志 |
| 2 | `SE_PKA_0_BUSY` | PKA 忙碌标志 |
| 3 | `SE_PKA_0_EN` | PKA 使能 |
| 7:4 | `SE_PKA_0_PROT_MD` | 保护模式选择 |
| 8 | `SE_PKA_0_INT` | 中断标志 |
| 9 | `SE_PKA_0_INT_CLR_1T` | 写 1 清除中断 |
| 10 | `SE_PKA_0_INT_SET` | 中断使能 |
| 11 | `SE_PKA_0_INT_MASK` | 中断屏蔽 |
| 12 | `SE_PKA_0_ENDIAN` | 端序配置 |
| 13 | `SE_PKA_0_RAM_CLR_MD` | RAM 清除模式 |
| 15 | `SE_PKA_0_STATUS_CLR_1T` | 写 1 清除状态 |
| 31:16 | `SE_PKA_0_STATUS` | PKA 状态码 |

```c
#define SEC_ENG_SE_PKA_0_DONE          (1 << 0U)
#define SEC_ENG_SE_PKA_0_DONE_CLR_1T   (1 << 1U)
#define SEC_ENG_SE_PKA_0_BUSY          (1 << 2U)
#define SEC_ENG_SE_PKA_0_EN            (1 << 3U)
#define SEC_ENG_SE_PKA_0_PROT_MD_SHIFT (4U)
#define SEC_ENG_SE_PKA_0_PROT_MD_MASK  (0xf << 4U)
#define SEC_ENG_SE_PKA_0_INT           (1 << 8U)
#define SEC_ENG_SE_PKA_0_INT_CLR_1T    (1 << 9U)
#define SEC_ENG_SE_PKA_0_INT_SET       (1 << 10U)
#define SEC_ENG_SE_PKA_0_INT_MASK      (1 << 11U)
#define SEC_ENG_SE_PKA_0_ENDIAN        (1 << 12U)
#define SEC_ENG_SE_PKA_0_RAM_CLR_MD    (1 << 13U)
#define SEC_ENG_SE_PKA_0_STATUS_CLR_1T (1 << 15U)
```

### PKA Control Register 1 (0x310)

| Bit(s) | Field | Description |
|--------|-------|-------------|
| 2:0 | `SE_PKA_0_HBURST` | AHB 突发传输长度 |
| 3 | `SE_PKA_0_HBYPASS` | AHB 旁路模式 |

---

## Architecture Notes

- **实现方式:** DSA 功能通过预编译库 `libpka_bl616.a` 实现，无公开 .c 源码
- **硬件依赖:** 依赖 SEC_ENG 中的 PKA 加速器 (`SEC_ENG_BASE + 0x300`)
- **ROM API:** 部分芯片支持 `romapi_bflb_sec_dsa_*` 快速路径
- **CRT 优化:** 建议填充 `crtCfg` 以启用 CRT 加速，可显著提升私钥签名性能
- **密钥管理:** 所有密钥数据以 32-bit word 数组形式传入，调用者负责安全存储
