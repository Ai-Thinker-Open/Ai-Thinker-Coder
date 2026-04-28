# SEC_ECDSA API Reference (BL616/BL618)

> **Source:** `bouffalo_sdk/drivers/lhal/include/bflb_sec_ecdsa.h`  
> **Implementation:** `bouffalo_sdk/drivers/lhal/src/pka/libpka_bl616.a` (预编译 PKA 库)  
> **Hardware:** SEC_ENG PKA (Public Key Accelerator) @ `0x20004300`  
> **Register Header:** `bouffalo_sdk/drivers/lhal/include/hardware/sec_eng_reg.h`

## Overview

SEC_ECDSA 模块提供基于 **ECDSA (Elliptic Curve Digital Signature Algorithm)** 的椭圆曲线数字签名和验证功能，以及 **ECDH (Elliptic Curve Diffie-Hellman)** 密钥交换功能。该模块利用 SEC_ENG 中的 **PKA (Public Key Accelerator)** 硬件加速器执行椭圆曲线点乘运算。

**主要特性：**
- ECDSA 签名生成 (`bflb_sec_ecdsa_sign`)
- ECDSA 签名验证 (`bflb_sec_ecdsa_verify`)
- ECDH 密钥交换 (`bflb_sec_ecdh_get_encrypt_key`)
- 密钥对生成与管理 (`bflb_sec_ecdsa_get_private_key`, `bflb_sec_ecdsa_get_public_key`)
- 硬件随机数生成 (`bflb_sec_ecc_get_random_value`)
- 支持多种标准椭圆曲线
- 基于 `libpka_bl616.a` 预编译库实现

## Base Address

| Peripheral | Offset | Address |
|------------|--------|---------|
| SEC_ENG_BASE | — | `0x20004000` |
| SEC_ENG PKA | `0x300` | `0x20004300` |

---

## Elliptic Curve Definitions

```c
#define ECP_SECP256R1 0    // NIST P-256 / secp256r1
#define ECP_SECP256K1 1    // secp256k1 (Bitcoin curve)
#define ECP_SECP384R1 2    // NIST P-384 / secp384r1 (需 ECP_SUPPORT_384=1)
```

| Macro | Value | Curve | Key Size |
|-------|-------|-------|----------|
| `ECP_SECP256R1` | 0 | NIST P-256 | 256-bit |
| `ECP_SECP256K1` | 1 | secp256k1 | 256-bit |
| `ECP_SECP384R1` | 2 | NIST P-384 | 384-bit |

> **注意:** `ECP_SECP384R1` 仅在启用 `ECP_SUPPORT_384` (定义为 1) 时可用。

---

## Configuration Structures

### bflb_ecdsa_s

ECDSA 操作句柄，持有密钥指针和曲线选择。

```c
struct bflb_ecdsa_s {
    uint8_t ecpId;           // 椭圆曲线 ID (ECP_SECP256R1 / ECP_SECP256K1 / ECP_SECP384R1)
    uint8_t pad[3];          // 对齐填充
    uint32_t *privateKey;    // 私钥指针 (32-bit word 数组)
    uint32_t *publicKeyx;    // 公钥 X 坐标指针
    uint32_t *publicKeyy;    // 公钥 Y 坐标指针
};
```

| Field | Type | Description |
|-------|------|-------------|
| `ecpId` | `uint8_t` | 椭圆曲线 ID |
| `pad[3]` | `uint8_t[3]` | 内存对齐填充 |
| `privateKey` | `uint32_t *` | 私钥缓冲区指针 |
| `publicKeyx` | `uint32_t *` | 公钥 X 坐标缓冲区指针 |
| `publicKeyy` | `uint32_t *` | 公钥 Y 坐标缓冲区指针 |

### bflb_ecdh_s

ECDH 密钥交换句柄。

```c
struct bflb_ecdh_s {
    uint8_t ecpId;    // 椭圆曲线 ID
};
```

| Field | Type | Description |
|-------|------|-------------|
| `ecpId` | `uint8_t` | 椭圆曲线 ID |

---

## LHAL API Functions

### ECDSA Functions

#### bflb_sec_ecdsa_init

初始化 ECDSA 句柄，分配 PKA 硬件资源。

```c
int bflb_sec_ecdsa_init(struct bflb_ecdsa_s *handle, uint8_t id);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `struct bflb_ecdsa_s *` | ECDSA 句柄指针 |
| `id` | `uint8_t` | 椭圆曲线 ID (`ECP_SECP256R1` 等) |

**Returns:** `0` 成功，负值表示错误

---

#### bflb_sec_ecdsa_deinit

释放 ECDSA 句柄占用的 PKA 硬件资源。

```c
int bflb_sec_ecdsa_deinit(struct bflb_ecdsa_s *handle);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `struct bflb_ecdsa_s *` | ECDSA 句柄指针 |

**Returns:** `0` 成功，负值表示错误

---

#### bflb_sec_ecdsa_sign

使用 ECDSA 私钥对消息哈希进行签名。

```c
int bflb_sec_ecdsa_sign(struct bflb_ecdsa_s *handle, const uint32_t *random_k,
                         const uint32_t *hash, uint32_t hashLenInWord,
                         uint32_t *r, uint32_t *s);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `struct bflb_ecdsa_s *` | 已初始化的 ECDSA 句柄 |
| `random_k` | `const uint32_t *` | ECDSA 签名所需的随机数 k |
| `hash` | `const uint32_t *` | 消息哈希值（32-bit word 数组） |
| `hashLenInWord` | `uint32_t` | 哈希长度（以 32-bit word 为单位） |
| `r` | `uint32_t *` | 输出: 签名 r 分量 |
| `s` | `uint32_t *` | 输出: 签名 s 分量 |

**Returns:** `0` 成功，负值表示错误

**说明:** `random_k` 必须为安全随机数，每次签名必须唯一。可使用 `bflb_sec_ecc_get_random_value()` 生成。

---

#### bflb_sec_ecdsa_verify

使用 ECDSA 公钥验证签名。

```c
int bflb_sec_ecdsa_verify(struct bflb_ecdsa_s *handle, const uint32_t *hash,
                           uint32_t hashLen, const uint32_t *r, const uint32_t *s);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `struct bflb_ecdsa_s *` | 已初始化的 ECDSA 句柄 |
| `hash` | `const uint32_t *` | 消息哈希值 |
| `hashLen` | `uint32_t` | 哈希长度（以 32-bit word 为单位） |
| `r` | `const uint32_t *` | 签名 r 分量 |
| `s` | `const uint32_t *` | 签名 s 分量 |

**Returns:** `0` 验证通过，负值表示验证失败

---

#### bflb_sec_ecdsa_get_private_key

获取或生成 ECDSA 私钥。

```c
int bflb_sec_ecdsa_get_private_key(struct bflb_ecdsa_s *handle, uint32_t *private_key);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `struct bflb_ecdsa_s *` | 已初始化的 ECDSA 句柄 |
| `private_key` | `uint32_t *` | 输出缓冲区（接收私钥） |

**Returns:** `0` 成功

---

#### bflb_sec_ecdsa_get_public_key

根据私钥计算 ECDSA 公钥。

```c
int bflb_sec_ecdsa_get_public_key(struct bflb_ecdsa_s *handle,
                                   const uint32_t *private_key,
                                   const uint32_t *pRx, const uint32_t *pRy);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `struct bflb_ecdsa_s *` | 已初始化的 ECDSA 句柄 |
| `private_key` | `const uint32_t *` | 私钥 |
| `pRx` | `const uint32_t *` | 输出: 公钥 X 坐标 |
| `pRy` | `const uint32_t *` | 输出: 公钥 Y 坐标 |

**Returns:** `0` 成功

> **注意:** 参数声明为 `const uint32_t *` 但实际用于输出。在调用时传入可写缓冲区。

---

### ECDH Functions

#### bflb_sec_ecdh_init

初始化 ECDH 密钥交换句柄。

```c
int bflb_sec_ecdh_init(struct bflb_ecdh_s *handle, uint8_t id);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `struct bflb_ecdh_s *` | ECDH 句柄指针 |
| `id` | `uint8_t` | 椭圆曲线 ID |

**Returns:** `0` 成功

---

#### bflb_sec_ecdh_deinit

释放 ECDH 句柄资源。

```c
int bflb_sec_ecdh_deinit(struct bflb_ecdh_s *handle);
```

**Returns:** `0` 成功

---

#### bflb_sec_ecdh_get_encrypt_key

执行 ECDH 密钥交换，计算共享密钥。

```c
int bflb_sec_ecdh_get_encrypt_key(struct bflb_ecdh_s *handle,
                                   const uint32_t *pkX, const uint32_t *pkY,
                                   const uint32_t *private_key,
                                   const uint32_t *pRx, const uint32_t *pRy);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `struct bflb_ecdh_s *` | ECDH 句柄 |
| `pkX` | `const uint32_t *` | 对方公钥 X 坐标 |
| `pkY` | `const uint32_t *` | 对方公钥 Y 坐标 |
| `private_key` | `const uint32_t *` | 己方私钥 |
| `pRx` | `const uint32_t *` | 输出: 共享密钥 X 坐标 |
| `pRy` | `const uint32_t *` | 输出: 共享密钥 Y 坐标 |

**Returns:** `0` 成功

---

#### bflb_sec_ecdh_get_public_key

根据私钥计算 ECDH 公钥。

```c
int bflb_sec_ecdh_get_public_key(struct bflb_ecdh_s *handle,
                                  const uint32_t *private_key,
                                  const uint32_t *pRx, const uint32_t *pRy);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `handle` | `struct bflb_ecdh_s *` | ECDH 句柄 |
| `private_key` | `const uint32_t *` | 私钥 |
| `pRx` | `const uint32_t *` | 输出: 公钥 X 坐标 |
| `pRy` | `const uint32_t *` | 输出: 公钥 Y 坐标 |

**Returns:** `0` 成功

---

### Random Number Generation

#### bflb_sec_ecc_get_random_value

生成 ECC 安全随机数（用于签名随机数 k 等）。

```c
int bflb_sec_ecc_get_random_value(uint32_t *data, uint32_t *max_ref, uint32_t size);
```

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `data` | `uint32_t *` | 输出随机数缓冲区 |
| `max_ref` | `uint32_t *` | 最大值参考（生成的随机数 < max_ref） |
| `size` | `uint32_t` | 数据大小（32-bit word 数） |

**Returns:** `0` 成功

**说明:** 该函数用于生成 ECDSA 签名所需的随机数 k。通常 `max_ref` 设为椭圆曲线的阶 n。

---

## Usage Examples

### Example 1: ECDSA 签名与验证 (secp256r1)

```c
#include "bflb_sec_ecdsa.h"
#include "bflb_sha.h"

void ecdsa_sign_verify_example(void)
{
    struct bflb_ecdsa_s ecdsa;
    int ret;

    // P-256: 私钥/公钥各 256-bit = 8 个 32-bit word
    uint32_t private_key[8] = {0};
    uint32_t public_key_x[8] = {0};
    uint32_t public_key_y[8] = {0};

    // 初始化 ECDSA (secp256r1)
    ret = bflb_sec_ecdsa_init(&ecdsa, ECP_SECP256R1);
    if (ret != 0) {
        printf("ECDSA init failed: %d\n", ret);
        return;
    }

    // 生成密钥对
    ret = bflb_sec_ecdsa_get_private_key(&ecdsa, private_key);
    if (ret != 0) {
        printf("Generate private key failed\n");
        return;
    }

    ret = bflb_sec_ecdsa_get_public_key(&ecdsa, private_key, public_key_x, public_key_y);
    if (ret != 0) {
        printf("Generate public key failed\n");
        return;
    }

    // 设置密钥到句柄
    ecdsa.privateKey = private_key;
    ecdsa.publicKeyx = public_key_x;
    ecdsa.publicKeyy = public_key_y;

    // 准备消息哈希 (SHA-256 = 8 words)
    uint8_t message[] = "ECDSA test message";
    uint32_t hash[8] = {0};
    // bflb_sha256(hash, message, sizeof(message));

    // 生成签名随机数 k
    // 注意: secp256r1 的阶 n 需从标准文档获取，此处为示意
    uint32_t n_order[8] = {
        0xFFFFFFFF, 0x00000000, 0xFFFFFFFF, 0xFFFFFFFF,
        0xBCE6FAAD, 0xA7179E84, 0xF3B9CAC2, 0xFC632551
    };
    uint32_t random_k[8] = {0};

    ret = bflb_sec_ecc_get_random_value(random_k, n_order, 8);
    if (ret != 0) {
        printf("Random generation failed\n");
        return;
    }

    // ECDSA 签名
    uint32_t r[8] = {0};
    uint32_t s[8] = {0};

    ret = bflb_sec_ecdsa_sign(&ecdsa, random_k, hash, 8, r, s);
    if (ret == 0) {
        printf("ECDSA sign success\n");
        printf("r: 0x%08X...\n", r[0]);
        printf("s: 0x%08X...\n", s[0]);
    } else {
        printf("ECDSA sign failed: %d\n", ret);
    }

    // ECDSA 验证
    ret = bflb_sec_ecdsa_verify(&ecdsa, hash, 8, r, s);
    if (ret == 0) {
        printf("ECDSA verify PASSED\n");
    } else {
        printf("ECDSA verify FAILED: %d\n", ret);
    }

    // 释放资源
    bflb_sec_ecdsa_deinit(&ecdsa);
}
```

### Example 2: ECDH 密钥交换

```c
#include "bflb_sec_ecdsa.h"

void ecdh_key_exchange_example(void)
{
    struct bflb_ecdh_s ecdh_a;  // A 方
    struct bflb_ecdh_s ecdh_b;  // B 方
    int ret;

    // P-256: 256-bit keys = 8 words
    uint32_t priv_a[8] = {0}, pub_ax[8] = {0}, pub_ay[8] = {0};
    uint32_t priv_b[8] = {0}, pub_bx[8] = {0}, pub_by[8] = {0};
    uint32_t shared_ax[8] = {0}, shared_ay[8] = {0};
    uint32_t shared_bx[8] = {0}, shared_by[8] = {0};

    // 初始化双方 ECDH
    bflb_sec_ecdh_init(&ecdh_a, ECP_SECP256R1);
    bflb_sec_ecdh_init(&ecdh_b, ECP_SECP256R1);

    // A 方生成密钥对
    bflb_sec_ecc_get_random_value(priv_a, (uint32_t[8]){
        0xFFFFFFFF, 0x00000000, 0xFFFFFFFF, 0xFFFFFFFF,
        0xBCE6FAAD, 0xA7179E84, 0xF3B9CAC2, 0xFC632551
    }, 8);
    bflb_sec_ecdh_get_public_key(&ecdh_a, priv_a, pub_ax, pub_ay);

    // B 方生成密钥对
    bflb_sec_ecc_get_random_value(priv_b, (uint32_t[8]){
        0xFFFFFFFF, 0x00000000, 0xFFFFFFFF, 0xFFFFFFFF,
        0xBCE6FAAD, 0xA7179E84, 0xF3B9CAC2, 0xFC632551
    }, 8);
    bflb_sec_ecdh_get_public_key(&ecdh_b, priv_b, pub_bx, pub_by);

    // ECDH: A 方用 B 的公钥 + A 的私钥计算共享密钥
    ret = bflb_sec_ecdh_get_encrypt_key(&ecdh_a, pub_bx, pub_by,
                                         priv_a, shared_ax, shared_ay);
    if (ret == 0) printf("A: ECDH shared key computed\n");

    // ECDH: B 方用 A 的公钥 + B 的私钥计算共享密钥
    ret = bflb_sec_ecdh_get_encrypt_key(&ecdh_b, pub_ax, pub_ay,
                                         priv_b, shared_bx, shared_by);
    if (ret == 0) printf("B: ECDH shared key computed\n");

    // 验证共享密钥一致 (shared_ax == shared_bx)
    bool match = (memcmp(shared_ax, shared_bx, 32) == 0);
    printf("Shared key match: %s\n", match ? "YES" : "NO");

    // 释放
    bflb_sec_ecdh_deinit(&ecdh_a);
    bflb_sec_ecdh_deinit(&ecdh_b);
}
```

### Example 3: 使用 secp256k1 (Bitcoin 曲线)

```c
void ecdsa_secp256k1_example(void)
{
    struct bflb_ecdsa_s ecdsa;

    bflb_sec_ecdsa_init(&ecdsa, ECP_SECP256K1);

    uint32_t private_key[8] = {0};
    uint32_t public_key_x[8] = {0};
    uint32_t public_key_y[8] = {0};

    bflb_sec_ecdsa_get_private_key(&ecdsa, private_key);
    bflb_sec_ecdsa_get_public_key(&ecdsa, private_key, public_key_x, public_key_y);

    // secp256k1 阶 n
    uint32_t n_order[8] = {
        0xFFFFFFFF, 0xFFFFFFFF, 0xFFFFFFFF, 0xFFFFFFFE,
        0xBAAEDCE6, 0xAF48A03B, 0xBFD25E8C, 0xD0364141
    };

    // 签名...
    // ...

    bflb_sec_ecdsa_deinit(&ecdsa);
}
```

---

## Register-Level Reference

ECDSA/ECDH 操作通过 SEC_ENG PKA 子系统完成。

### SEC_ENG Architecture

| Block | Offset | Purpose |
|-------|--------|---------|
| SHA | `0x000-0x0FC` | SHA-1/224/256/384/512 |
| AES | `0x100-0x1FC` | AES-128/192/256 ECB/CTR/CBC/XTS |
| TRNG | `0x200-0x2FC` | True Random Number Generator |
| **PKA** | **`0x300-0x3FC`** | **Public Key Accelerator (DSA/ECDSA/ECDH)** |
| CDET | `0x400-0x4FC` | Clock Detection |
| GMAC | `0x500-0x5FC` | Ethernet GMAC |

### PKA Register Offsets

| Offset | Register | Description |
|--------|----------|-------------|
| `0x300` | `SE_PKA_0_CTRL_0` | PKA 控制寄存器 0 (使能/状态/中断) |
| `0x30C` | `SE_PKA_0_SEED` | PKA 随机种子 |
| `0x310` | `SE_PKA_0_CTRL_1` | PKA 控制寄存器 1 (AHB 突发/旁路) |
| `0x340` | `SE_PKA_0_RW` | PKA 数据读写端口 |
| `0x360` | `SE_PKA_0_RW_BURST` | PKA 批量数据读写端口 |
| `0x3FC` | `SE_PKA_0_CTRL_PROT` | PKA 访问保护 |

### PKA Control Register 0 (0x300)

| Bit(s) | Field | Description |
|--------|-------|-------------|
| 0 | `DONE` | 操作完成标志 (RO) |
| 1 | `DONE_CLR_1T` | 写 1 清除完成标志 |
| 2 | `BUSY` | PKA 忙碌标志 (RO) |
| 3 | `EN` | PKA 使能 |
| 7:4 | `PROT_MD` | 保护模式 (0x0-0xF) |
| 8 | `INT` | 中断标志 |
| 9 | `INT_CLR_1T` | 写 1 清除中断 |
| 10 | `INT_SET` | 中断使能位 |
| 11 | `INT_MASK` | 中断屏蔽 |
| 12 | `ENDIAN` | 端序配置 |
| 13 | `RAM_CLR_MD` | PKA RAM 清除模式 |
| 15 | `STATUS_CLR_1T` | 写 1 清除状态 |
| 31:16 | `STATUS` | PKA 操作状态码 |

### PKA Control Register 1 (0x310)

| Bit(s) | Field | Description |
|--------|-------|-------------|
| 2:0 | `HBURST` | AHB 突发传输长度 (0-7) |
| 3 | `HBYPASS` | AHB 旁路模式 |

---

## Key Size Reference

| Curve | Private Key | Public Key (X) | Public Key (Y) | Hash (SHA) |
|-------|-------------|----------------|---------------|------------|
| secp256r1 (P-256) | 8 words (256-bit) | 8 words | 8 words | SHA-256 = 8 words |
| secp256k1 | 8 words (256-bit) | 8 words | 8 words | SHA-256 = 8 words |
| secp384r1 (P-384) | 12 words (384-bit) | 12 words | 12 words | SHA-384 = 12 words |

## Architecture Notes

- **实现方式:** ECDSA/ECDH 功能通过预编译库 `libpka_bl616.a` 实现，无公开 .c 源码
- **硬件依赖:** 依赖 SEC_ENG 中的 PKA 加速器 (`SEC_ENG_BASE + 0x300`)
- **随机数安全:** `bflb_sec_ecc_get_random_value()` 利用 SEC_ENG 的 TRNG (True Random Number Generator) 生成安全随机数
- **ECDSA 签名随机数 k:** 每次签名必须使用唯一的随机数 k，泄露或重复使用 k 将导致私钥泄露
- **ECDSA vs ECDH:** ECDSA 用于签名/验证，ECDH 用于密钥交换，分别使用不同的句柄类型
- **资源管理:** 使用完毕后必须调用 `deinit` 释放 PKA 硬件资源
