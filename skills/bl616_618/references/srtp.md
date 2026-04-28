# libSRTP2 技术参考文档

## 概述

libSRTP2 是一个开源的 SRTP（Secure Real-time Transport Protocol，安全实时传输协议）库，提供 RTP/RTCP 数据的加密、认证和完整性保护。该库由 Cisco Systems 开发，广泛应用于 VoIP、视频会议、WebRTC 等实时通信场景。

libSRTP2 支持两种主要的加密模式：

- **AES-CM（AES Counter Mode）**：基于计数模式的 AES 加密，配合 HMAC-SHA1 进行消息认证
- **AES-GCM（AES Galois Counter Mode）**：AEAD（Authenticated Encryption with Associated Data）模式，同时完成加密和认证

## 关键常量

libSRTP2 定义了以下核心常量：

| 常量名称 | 值 | 说明 |
|---------|-----|------|
| `SRTP_MASTER_KEY_LEN` | 30 | 母钥（Master Key）的名义长度 |
| `SRTP_MAX_KEY_LEN` | 64 | 支持的最大密钥长度（字节） |
| `SRTP_MAX_TAG_LEN` | 16 | 最大认证标签长度（字节） |
| `SRTP_MAX_TRAILER_LEN` | 144 | 最大尾部扩展长度（含标签和 MKI） |
| `SRTP_SALT_LEN` | 14 | 密钥盐值长度（字节） |
| `SRTP_AEAD_SALT_LEN` | 12 | GCM 模式盐值长度（字节） |

### AES 密钥长度定义

```c
#define SRTP_AES_128_KEY_LEN 16    // 128-bit AES 密钥
#define SRTP_AES_192_KEY_LEN 24    // 192-bit AES 密钥
#define SRTP_AES_256_KEY_LEN 32    // 256-bit AES 密钥

// 包含盐值的完整密钥长度
#define SRTP_AES_ICM_128_KEY_LEN_WSALT 30  // 14 + 16
#define SRTP_AES_ICM_256_KEY_LEN_WSALT 46  // 14 + 32
#define SRTP_AES_GCM_128_KEY_LEN_WSALT 28  // 12 + 16
#define SRTP_AES_GCM_256_KEY_LEN_WSALT 44  // 12 + 32
```

## 错误码

libSRTP2 使用 `srtp_err_status_t` 枚举定义所有错误状态：

```c
typedef enum {
    srtp_err_status_ok = 0,             // 操作成功
    srtp_err_status_fail = 1,           // 未指定的失败
    srtp_err_status_bad_param = 2,      // 不支持的参数
    srtp_err_status_alloc_fail = 3,      // 内存分配失败
    srtp_err_status_dealloc_fail = 4,    // 内存释放失败
    srtp_err_status_init_fail = 5,       // 初始化失败
    srtp_err_status_terminus = 6,        // 数据长度不足
    srtp_err_status_auth_fail = 7,       // 认证失败
    srtp_err_status_cipher_fail = 8,     // 加密失败
    srtp_err_status_replay_fail = 9,     // 重放攻击检测失败（索引错误）
    srtp_err_status_replay_old = 10,     // 重放攻击检测失败（索引过旧）
    srtp_err_status_algo_fail = 11,      // 算法测试失败
    srtp_err_status_no_such_op = 12,      // 不支持的操作
    srtp_err_status_no_ctx = 13,          // 未找到合适的上下文
    srtp_err_status_key_expired = 15,     // 密钥已过期
    srtp_err_status_bad_mki = 25,         // MKI 无效
    srtp_err_status_pkt_idx_old = 26,     // 数据包索引过旧
    srtp_err_status_pkt_idx_adv = 27      // 数据包索引提前，需要重置
} srtp_err_status_t;
```

## 核心数据结构

### srtp_crypto_policy_t

加密策略结构体，定义加密和认证的具体参数：

```c
typedef struct srtp_crypto_policy_t {
    srtp_cipher_type_id_t cipher_type;  // 加密算法类型
    int cipher_key_len;                  // 加密密钥长度
    srtp_auth_type_id_t auth_type;       // 认证算法类型
    int auth_key_len;                    // 认证密钥长度
    int auth_tag_len;                    // 认证标签长度
    srtp_sec_serv_t sec_serv;           // 安全服务标志
} srtp_crypto_policy_t;
```

### SSRC 类型

```c
typedef enum {
    ssrc_undefined = 0,    // 未定义的 SSRC
    ssrc_specific = 1,     // 特定的 SSRC 值
    ssrc_any_inbound = 2, // 任意入站 SSRC（用于 srtp_unprotect）
    ssrc_any_outbound = 3 // 任意出站 SSRC（用于 srtp_protect）
} srtp_ssrc_type_t;
```

### srtp_policy_t

SRTP 会话策略结构体，定义单个 SRTP 流的策略：

```c
typedef struct srtp_policy_t {
    srtp_ssrc_t ssrc;               // SSRC 值或通配符类型
    srtp_crypto_policy_t rtp;        // RTP 加密策略
    srtp_crypto_policy_t rtcp;      // RTCP 加密策略
    unsigned char *key;              // 主密钥指针
    srtp_master_key_t **keys;       // 多组主密钥数组
    unsigned long num_master_keys;   // 主密钥数量
    srtp_ekt_policy_t ekt;          // EKT 策略指针
    unsigned long window_size;       // 重放保护窗口大小
    int allow_repeat_tx;            // 是否允许重复传输
    int *enc_xtn_hdr;               // 需要加密的扩展头 ID 列表
    int enc_xtn_hdr_count;          // 扩展头数量
    struct srtp_policy_t *next;     // 指向下一个策略的指针
} srtp_policy_t;
```

### srtp_t（srtp_session_t）

会话句柄类型，指向内部会话上下文的指针：

```c
typedef srtp_ctx_t *srtp_t;  // SRTP 会话句柄
```

## 加密套件

libSRTP2 支持多种预定义的加密套件，通过以下函数初始化：

### AES-CM 系列

| 加密套件 | 函数 | 说明 |
|---------|------|------|
| AES_CM_128_HMAC_SHA1_80 | `srtp_crypto_policy_set_aes_cm_128_hmac_sha1_80()` | AES-128 计数器模式 + HMAC-SHA1-80 位认证 |
| AES_CM_128_HMAC_SHA1_32 | `srtp_crypto_policy_set_aes_cm_128_hmac_sha1_32()` | AES-128 + HMAC-SHA1-32 位认证（仅用于 RTP） |
| AES_CM_256_HMAC_SHA1_80 | `srtp_crypto_policy_set_aes_cm_256_hmac_sha1_80()` | AES-256 + HMAC-SHA1-80 位认证 |
| AES_CM_256_HMAC_SHA1_32 | `srtp_crypto_policy_set_aes_cm_256_hmac_sha1_32()` | AES-256 + HMAC-SHA1-32 位认证 |
| AES_CM_192_HMAC_SHA1_80 | `srtp_crypto_policy_set_aes_cm_192_hmac_sha1_80()` | AES-192 + HMAC-SHA1-80 位认证 |

### AES-GCM 系列

| 加密套件 | 函数 | 说明 |
|---------|------|------|
| AES_GCM_128_16 | `srtp_crypto_policy_set_aes_gcm_128_16_auth()` | AES-128-GCM + 16 字节认证标签 |
| AES_GCM_256_16 | `srtp_crypto_policy_set_aes_gcm_256_16_auth()` | AES-256-GCM + 16 字节认证标签 |
| AES_GCM_128_8 | `srtp_crypto_policy_set_aes_gcm_128_8_auth()` | AES-128-GCM + 8 字节认证标签 |
| AES_GCM_256_8 | `srtp_crypto_policy_set_aes_gcm_256_8_auth()` | AES-256-GCM + 8 字节认证标签 |

### 安全服务标志

```c
typedef enum {
    sec_serv_none = 0,         // 不启用任何安全服务
    sec_serv_conf = 1,         // 仅加密（机密性）
    sec_serv_auth = 2,        // 仅认证
    sec_serv_conf_and_auth = 3 // 加密 + 认证
} srtp_sec_serv_t;
```

## ROC（Roll-over Counter）同步机制

ROC 是 SRTP 的核心同步机制，用于处理 RTP 序列号（16 位）的回绕问题。

### 工作原理

1. **序列号回绕**：RTP 序列号只有 16 位（0-65535），当超过 65535 时会回绕到 0
2. **ROC 扩展**：SRTP 使用 32 位的 ROC 来扩展 16 位序列号，形成完整的 48 位包索引
3. **同步条件**：当接收到的序列号比上一次小超过 32768（2^15）时，说明序列号已回绕，ROC 需要加 1

### ROC 维护

```c
// ROC 更新逻辑示例
if (incoming_seq > last_seq) {
    if (incoming_seq - last_seq > 32768) {
        roc--;  // 序列号回绕，向后
    } else {
        // 正常顺序
    }
} else {
    if (last_seq - incoming_seq > 32768) {
        roc++;  // 序列号回绕，向前
    }
}
```

### 重放保护

libSRTP2 提供基于窗口的重放保护机制：

- `window_size`：定义重放检测窗口大小，默认 128 位
- 早于窗口左界的包将被拒绝
- 支持配置 `allow_repeat_tx` 允许相同序列号重传

## 核心 API

### 库初始化与销毁

```c
// 初始化 SRTP 库（必须在调用其他 SRTP 函数之前调用）
srtp_err_status_t srtp_init(void);

// 关闭 SRTP 库（所有 SRTP 操作完成后调用）
srtp_err_status_t srtp_shutdown(void);
```

### 会话创建与销毁

```c
// 创建 SRTP 会话
// session: 输出参数，返回会话句柄
// policy: 输入参数，会话策略配置
srtp_err_status_t srtp_create(srtp_t *session, const srtp_policy_t *policy);

// 释放 SRTP 会话
srtp_err_status_t srtp_dealloc(srtp_t s);
```

### 添加/移除流

```c
// 在现有会话中添加新的 SRTP 流
srtp_err_status_t srtp_add_stream(srtp_t session, const srtp_policy_t *policy);

// 从会话中移除指定 SSRC 的流
srtp_err_status_t srtp_remove_stream(srtp_t session, uint32_t ssrc);

// 更新会话中所有流的密钥（保留 ROC 值）
srtp_err_status_t srtp_update(srtp_t session, const srtp_policy_t *policy);

// 更新指定流的密钥
srtp_err_status_t srtp_update_stream(srtp_t session, const srtp_policy_t *policy);
```

### RTP 保护

```c
// 保护 RTP 包（发送端）
// ctx: 会话句柄
// rtp_hdr: 输入的 RTP 包，输出为 SRTP 包
// len_ptr: 输入包长度，输出保护后长度
srtp_err_status_t srtp_protect(srtp_t ctx, void *rtp_hdr, int *len_ptr);

// 使用 MKI 的保护版本
srtp_err_status_t srtp_protect_mki(srtp_ctx_t *ctx,
                                    void *rtp_hdr,
                                    int *pkt_octet_len,
                                    unsigned int use_mki,
                                    unsigned int mki_index);

// 解除保护（接收端）
// ctx: 会话句柄
// srtp_hdr: 输入的 SRTP 包，输出为 RTP 包
// len_ptr: 输入包长度，输出解包后长度
srtp_err_status_t srtp_unprotect(srtp_t ctx, void *srtp_hdr, int *len_ptr);

// 使用 MKI 的解包版本
srtp_err_status_t srtp_unprotect_mki(srtp_t ctx,
                                      void *srtp_hdr,
                                      int *len_ptr,
                                      unsigned int use_mki);
```

### RTCP 保护

```c
// 保护 RTCP 包
srtp_err_status_t srtp_protect_rtcp(srtp_t ctx, void *rtcp_hdr, int *len_ptr);

// 解除 RTCP 保护
srtp_err_status_t srtp_unprotect_rtcp(srtp_t ctx, void *srtcp_hdr, int *len_ptr);
```

### 策略辅助函数

```c
// 设置默认 RTP 策略（AES_CM_128_HMAC_SHA1_80）
void srtp_crypto_policy_set_rtp_default(srtp_crypto_policy_t *p);

// 设置默认 RTCP 策略
void srtp_crypto_policy_set_rtcp_default(srtp_crypto_policy_t *p);

// 从 SRTP Profile 设置策略
srtp_err_status_t srtp_crypto_policy_set_from_profile_for_rtp(
    srtp_crypto_policy_t *policy,
    srtp_profile_t profile);

srtp_err_status_t srtp_crypto_policy_set_from_profile_for_rtcp(
    srtp_crypto_policy_t *policy,
    srtp_profile_t profile);

// 获取 Profile 的主密钥长度
unsigned int srtp_profile_get_master_key_length(srtp_profile_t profile);

// 获取 Profile 的主盐值长度
unsigned int srtp_profile_get_master_salt_length(srtp_profile_t profile);
```

## 代码示例

### 基本使用流程

```c
#include "srtp.h"
#include <stdio.h>
#include <string.h>

#define RTP_PAYLOAD_LEN 160
#define RTP_HEADER_LEN 12

// RTP 包结构（简化）
typedef struct {
    uint8_t version_p_x_cc;    // V、P、X、CC 字段
    uint8_t m_pt;              // M 和 Payload Type
    uint16_t seq;               // 序列号（网络字节序）
    uint32_t ts;                // 时间戳
    uint32_t ssrc;              // SSRC
    uint8_t payload[RTP_PAYLOAD_LEN];
} rtp_packet_t;

// AES-GCM 256 加密示例
int example_aes_gcm_256()
{
    srtp_err_status_t err;
    srtp_t session;
    srtp_policy_t policy;

    // 母钥：30 字节（包含密钥和盐值）
    // GCM-256 模式：密钥 32 字节 + 盐值 12 字节 = 44 字节
    unsigned char key[SRTP_AES_GCM_256_KEY_LEN_WSALT] = {
        0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
        0x08, 0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f,
        0x10, 0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17,
        0x18, 0x19, 0x1a, 0x1b, 0x1c, 0x1d, 0x1e, 0x1f,
        // 盐值（用于 GCM IV）
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00
    };

    // 初始化库
    err = srtp_init();
    if (err != srtp_err_status_ok) {
        printf("srtp_init failed: %d\n", err);
        return -1;
    }

    // 初始化策略结构
    memset(&policy, 0, sizeof(policy));

    // 配置 SSRC（任意入站）
    policy.ssrc.type = ssrc_any_inbound;
    policy.ssrc.value = 0;

    // 设置 RTP 加密策略：AES-GCM-256 + 16 字节认证标签
    srtp_crypto_policy_set_aes_gcm_256_16_auth(&policy.rtp);

    // 设置 RTCP 加密策略
    srtp_crypto_policy_set_aes_gcm_256_16_auth(&policy.rtcp);

    // 设置密钥
    policy.key = key;

    // 设置重放保护窗口
    policy.window_size = 128;

    // 允许重传
    policy.allow_repeat_tx = 0;

    // 创建会话
    err = srtp_create(&session, &policy);
    if (err != srtp_err_status_ok) {
        printf("srtp_create failed: %d\n", err);
        return -1;
    }

    // 准备 RTP 包
    rtp_packet_t rtp_pkt;
    memset(&rtp_pkt, 0, sizeof(rtp_pkt));
    rtp_pkt.version_p_x_cc = 0x80;  // RTP 版本 2
    rtp_pkt.m_pt = 0x00;           // PT = 0（PCM u-law）
    rtp_pkt.seq = 1;               // 序列号
    rtp_pkt.ts = 160;              // 时间戳
    rtp_pkt.ssrc = 0x12345678;     // SSRC

    // 填充负载数据
    memset(rtp_pkt.payload, 0x55, RTP_PAYLOAD_LEN);

    // 保护 RTP 包
    int len = RTP_HEADER_LEN + RTP_PAYLOAD_LEN;
    err = srtp_protect(session, &rtp_pkt, &len);
    if (err != srtp_err_status_ok) {
        printf("srtp_protect failed: %d\n", err);
        srtp_dealloc(session);
        return -1;
    }

    printf("Protected RTP packet, new length: %d\n", len);
    printf("SRTP header + encrypted payload + auth tag (16 bytes)\n");

    // 解包 RTP 包
    int unprotect_len = len;
    err = srtp_unprotect(session, &rtp_pkt, &unprotect_len);
    if (err != srtp_err_status_ok) {
        printf("srtp_unprotect failed: %d\n", err);
        srtp_dealloc(session);
        return -1;
    }

    printf("Unprotected RTP packet, length: %d\n", unprotect_len);

    // 释放会话
    srtp_dealloc(session);

    return 0;
}

// AES-CM 128 加密示例
int example_aes_cm_128()
{
    srtp_err_status_t err;
    srtp_t session;
    srtp_policy_t policy;

    // 母钥：30 字节
    // CM 模式：密钥 16 字节 + 盐值 14 字节 = 30 字节
    unsigned char key[SRTP_MASTER_KEY_LEN] = {
        0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
        0x08, 0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f,
        // 盐值
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00
    };

    // 初始化库
    err = srtp_init();
    if (err != srtp_err_status_ok) {
        return -1;
    }

    // 初始化策略
    memset(&policy, 0, sizeof(policy));
    policy.ssrc.type = ssrc_any_outbound;
    policy.ssrc.value = 0;

    // 使用默认策略：AES-CM-128 + HMAC-SHA1-80
    srtp_crypto_policy_set_aes_cm_128_hmac_sha1_80(&policy.rtp);
    srtp_crypto_policy_set_aes_cm_128_hmac_sha1_80(&policy.rtcp);

    policy.key = key;
    policy.window_size = 128;
    policy.allow_repeat_tx = 0;

    // 创建会话
    err = srtp_create(&session, &policy);
    if (err != srtp_err_status_ok) {
        return -1;
    }

    // 准备并保护 RTP 包
    rtp_packet_t rtp_pkt = {0};
    rtp_pkt.version_p_x_cc = 0x80;
    rtp_pkt.m_pt = 0x60;  // PT = 96（动态类型）
    rtp_pkt.seq = 100;
    rtp_pkt.ts = 1000;
    rtp_pkt.ssrc = 0x87654321;

    int len = RTP_HEADER_LEN + RTP_PAYLOAD_LEN;
    err = srtp_protect(session, &rtp_pkt, &len);
    if (err != srtp_err_status_ok) {
        printf("Protect failed: %d\n", err);
        srtp_dealloc(session);
        return -1;
    }

    printf("Protected packet size: %d bytes (includes 10-byte auth tag)\n", len);

    // 解包验证
    int unprotect_len = len;
    err = srtp_unprotect(session, &rtp_pkt, &unprotect_len);
    if (err != srtp_err_status_ok) {
        printf("Unprotect failed: %d\n", err);
    }

    srtp_dealloc(session);
    return 0;
}

// 多流配置示例
int example_multi_stream()
{
    srtp_err_status_t err;
    srtp_t session;
    srtp_policy_t policy[3];  // 三个策略
    srtp_policy_t *policy_list = NULL;

    err = srtp_init();
    if (err != srtp_err_status_ok) {
        return -1;
    }

    // 流 1：SSRC = 0x11111111
    memset(&policy[0], 0, sizeof(policy[0]));
    policy[0].ssrc.type = ssrc_specific;
    policy[0].ssrc.value = 0x11111111;
    srtp_crypto_policy_set_aes_cm_128_hmac_sha1_80(&policy[0].rtp);
    policy[0].key = (unsigned char *)"0123456789abcdef0123456789abcdef";  // 30 字节
    policy[0].next = &policy[1];

    // 流 2：SSRC = 0x22222222
    memset(&policy[1], 0, sizeof(policy[1]));
    policy[1].ssrc.type = ssrc_specific;
    policy[1].ssrc.value = 0x22222222;
    srtp_crypto_policy_set_aes_gcm_128_16_auth(&policy[1].rtp);
    policy[1].key = (unsigned char *)"fedcba9876543210fedcba9876543210";  // 30 字节
    policy[1].next = &policy[2];

    // 流 3：通配符（匹配其他所有 SSRC）
    memset(&policy[2], 0, sizeof(policy[2]));
    policy[2].ssrc.type = ssrc_any_inbound;
    policy[2].ssrc.value = 0;
    srtp_crypto_policy_set_aes_cm_128_hmac_sha1_80(&policy[2].rtp);
    policy[2].key = (unsigned char *)"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa";  // 30 字节
    policy[2].next = NULL;  // 列表结束

    // 创建会话（传入策略链表）
    err = srtp_create(&session, policy);
    if (err != srtp_err_status_ok) {
        return -1;
    }

    printf("Multi-stream SRTP session created\n");

    srtp_dealloc(session);
    return 0;
}
```

### 密钥更新示例

```c
// 动态更新会话密钥
int example_key_update()
{
    srtp_err_status_t err;
    srtp_t session;
    srtp_policy_t policy;
    unsigned char new_key[SRTP_MASTER_KEY_LEN];

    // 初始化（省略...）
    // err = srtp_init();
    // err = srtp_create(&session, &policy);

    // 生成新密钥（实际应用中应使用安全的随机数生成器）
    memset(new_key, 0xAB, SRTP_MASTER_KEY_LEN);

    // 更新密钥（保留 ROC 值，实现无缝切换）
    memset(&policy, 0, sizeof(policy));
    policy.ssrc.type = ssrc_any_inbound;
    srtp_crypto_policy_set_aes_cm_128_hmac_sha1_80(&policy.rtp);
    policy.key = new_key;

    err = srtp_update(session, &policy);
    if (err != srtp_err_status_ok) {
        printf("Key update failed: %d\n", err);
        return -1;
    }

    printf("Session key updated successfully\n");
    return 0;
}
```

## AES-GCM 模式详解

AES-GCM 是一种 AEAD（Authenticated Encryption with Associated Data）算法，在单一加密操作中同时提供：

- **加密**：使用伽罗瓦计数器模式
- **认证**：使用 GMAC（Galois Message Authentication Code）

### AES-GCM 与 AES-CM 的区别

| 特性 | AES-CM + HMAC | AES-GCM |
|------|---------------|---------|
| 加密算法 | AES 计数器模式 | AES 伽罗瓦计数器模式 |
| 认证算法 | HMAC-SHA1（单独计算） | GMAC（内嵌认证） |
| 认证标签位置 | 尾部 | 尾部 |
| 额外数据（AD） | 需要单独处理 | 内置支持 |
| 计算效率 | 需要两次调用 | 一次调用完成 |
| 典型标签长度 | 80 位（10 字节）或 32 位 | 128 位（16 字节）或 64 位（8 字节） |

### GCM 密钥结构

```c
// AES-GCM-128 完整密钥：16 字节密钥 + 12 字节盐值 = 28 字节
typedef struct {
    uint8_t key[16];    // AES 密钥
    uint8_t salt[12];   // GCM IV 盐值
} aes_gcm_128_key_t;

// AES-GCM-256 完整密钥：32 字节密钥 + 12 字节盐值 = 44 字节
typedef struct {
    uint8_t key[32];    // AES 密钥
    uint8_t salt[12];   // GCM IV 盐值
} aes_gcm_256_key_t;
```

### GCM IV 构造

SRTP 中 GCM 模式的 IV（初始化向量）构造方式：

```
IV = (salt << 16) | ROC | SEQ
```

- `salt`：12 字节盐值
- `ROC`：32 位的回绕计数器
- `SEQ`：16 位的 RTP 序列号

## 线程安全性

libSRTP2 本身**不是线程安全的**，使用时需要注意：

1. **每个 SRTP 流独立**：不同的 `srtp_t` 实例可以并行使用
2. **同一会话多线程**：不推荐多个线程同时对同一个 `srtp_t` 调用保护/解保护函数
3. **线程隔离**：建议每个线程使用独立的 `srtp_t` 实例，或使用互斥锁保护

## 常见错误处理

```c
// 错误处理示例
srtp_err_status_t handle_srtp_error(srtp_err_status_t err)
{
    switch (err) {
        case srtp_err_status_ok:
            return err;
        case srtp_err_status_auth_fail:
            printf("Authentication failed - packet corrupted or tampered\n");
            break;
        case srtp_err_status_replay_fail:
            printf("Replay attack detected\n");
            break;
        case srtp_err_status_pkt_idx_old:
            printf("Packet index too old\n");
            break;
        case srtp_err_status_key_expired:
            printf("Session key expired\n");
            break;
        default:
            printf("SRTP error: %d\n", err);
            break;
    }
    return err;
}
```

## 性能考虑

1. **批处理**：对于高带宽应用，考虑批量处理多个包
2. **内存对齐**：`srtp_protect()` 假设数据在 32 位边界对齐
3. **缓冲区大小**：调用 `srtp_protect()` 前，确保缓冲区有 `SRTP_MAX_TRAILER_LEN` 的额外空间
4. **GCM vs CM**：GCM 模式通常比 CM + HMAC 组合更快（单次加密+认证）

## 参考

- **RFC 3711**：SRTP（Secure Real-time Transport Protocol）
- **RFC 6188**：Additional AES-CTR Modes for SRTP
- **RFC 7714**：AES-GCM Authenticated Encryption for SRTP
- **RFC 4568**：SDP Security Descriptions for Media Streams
- **IANA SRTP Protection Profile**：https://www.iana.org/assignments/srtp-protection/srtp-protection.xhtml
- **libSRTP 官方文档**：https://github.com/cisco/libsrtp
