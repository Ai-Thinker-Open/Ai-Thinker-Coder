# BL616/618 Matter Factory Data (MFD) 技术文档

## 概述

Matter 是 CSA（Connectivity Standards Alliance）联盟制定的智能家居互联互通标准，旨在统一不同生态系统的设备通信协议。BL616/618 作为支持 Matter 协议的无线 MCU，通过 **mfd**（Matter Factory Data）模块存储和管理设备认证所需的核心数据。

MFD 模块负责从芯片 Flash 的特定存储区域读取工厂预置的认证数据，包括设备证书、私钥、PAI 证书、CD（Certificate Declaration）等敏感信息。这些数据在设备配网（Commissioning）过程中被 Matter 协议栈调用，用于设备身份验证和加密通信。

## MFD 版本信息

MFD 模块版本为 **1.6.1**，定义如下：

```c
#define VERSION_MFD_MAJOR 1
#define VERSION_MFD_MINOR 6
#define VERSION_MFD_PATCH 1
```

## MFD 初始化

在使用任何 MFD 数据读取接口之前，必须调用 `mfd_init()` 完成工厂数据区的初始化：

```c
bool mfd_init(void);
```

`mfd_init()` 会执行以下操作：

1. 初始化 Flash 访问控制
2. 定位工厂数据区在 Flash 中的存储地址
3. 验证数据区域的完整性
4. 解密需要加密的敏感数据（如私钥）
5. 准备数据读取环境

返回值表示初始化是否成功。只有初始化成功后，才能正常调用其他 MFD 数据读取接口。

## 设备证书相关接口

### DAC 证书操作

设备认证证书（Device Attestation Certificate，DAC）是 Matter 设备的核心身份标识。

```c
int mfd_getDacCert(uint8_t *p, uint32_t size);
uint8_t *mfd_getDacCertPtr(uint32_t *psize);
```

- `mfd_getDacCert()`：将 DAC 证书复制到指定缓冲区，返回读取状态
- `mfd_getDacCertPtr()`：直接返回 DAC 证书在内存中的指针，psize 存储证书长度

### DAC 私钥操作

设备私钥用于证书签名验证，必须妥善保管：

```c
int mfd_getDacPrivateKey(uint8_t *p, uint32_t size);
uint8_t *mfd_getDacPrivateKeyPtr(uint32_t *psize);
```

私钥数据在 Flash 中以加密形式存储，`mfd_init()` 会完成解密操作。

### PAI 证书

PAI（Product Attestation Intermediate）证书是证书链中的中间层：

```c
int mfd_getPaiCert(uint8_t *p, uint32_t size);
```

### CD 证书声明

CD（Certificate Declaration）声明证书用于声明设备的认证状态：

```c
int mfd_getCd(uint8_t *p, uint32_t size);
```

## 设备配网参数

### 配对码与识别码

```c
int mfd_getPasscode(uint8_t *p, uint32_t size);
int mfd_getDiscriminator(uint8_t *p, uint32_t size);
```

- `mfd_getPasscode()`：获取设备配对码（8 位数字），用于 SPAKE2+ 认证
- `mfd_getDiscriminator()`：获取设备识别码，用于 BLE 广播和服务发现

### 旋转设备唯一标识

```c
int mfd_getRotatingDeviceIdUniqueId(uint8_t *p, uint32_t size);
```

旋转设备唯一标识用于生成设备rotate ID，增强设备识别隐私性。

## 厂商信息

### 厂商 ID 和名称

```c
int mfd_getVendorId(uint8_t *buf, uint32_t size);
int mfd_getVendorName(char *buf, uint32_t size);
```

Vendor ID 是由 CSA 分配的唯一标识符，Vendor Name 是可读的厂商名称字符串。

### 产品信息

```c
int mfd_getProductId(uint8_t *buf, uint32_t size);
int mfd_getProductName(char *buf, uint32_t size);
```

Product ID 和 Product Name 用于标识具体产品型号。

### 硬件版本

```c
int mfd_getHardwareVersion(uint8_t *buf, uint32_t size);
int mfd_getHardwareVersionString(char *buf, uint32_t size);
```

硬件版本信息用于固件兼容性和设备识别。

## SPAKE2+ 认证参数

SPAKE2+ 是一种基于密码的认证协议，用于在不可信网络上建立安全连接。Matter 协议使用 SPAKE2+ 实现设备配对时的身份验证。

### SPAKE2+ 迭代因子

```c
int mfd_getSapke2It(uint8_t *p, uint32_t size);
```

IT（Iterations）是 PBKDF 函数的迭代次数，用于增加暴力破解难度。

### SPAKE2+ Salt

```c
int mfd_getSapke2Salt(uint8_t *p, uint32_t size);
```

Salt 是随机生成的盐值，与迭代因子共同用于派生认证密钥。

### SPAKE2+ Verifier

```c
int mfd_getSapke2Verifier(uint8_t *p, uint32_t size);
```

Verifier 是基于用户配对码生成的验证器，由Commissioner和Device各自计算后比对完成相互认证。

## 通用数据访问接口

### Element ID 查询

MFD 提供基于 Element ID 的通用数据访问接口：

```c
int mfd_getElementById(int16_t id, uint8_t *buf, uint32_t size);
```

通过 Element ID 可以访问工厂数据区中的任意数据项。这种方式适合访问标准 MFD 接口未覆盖的扩展数据字段。

## DAC 证书链结构

Matter 设备采用三层证书链结构实现信任传递：

```
┌─────────────────┐
│      CD         │  Certificate Declaration（证书声明）
│  (自签名证书)    │  声明设备符合 Matter 认证要求
└────────┬────────┘
         │ 验证签名
         ▼
┌─────────────────┐
│      PAI        │  Product Attestation Intermediate（产品认证中间证书）
│  (CSA 签发)     │  由 CSA 根证书签发
└────────┬────────┘
         │ 验证签名
         ▼
┌─────────────────┐
│      DAC        │  Device Attestation Certificate（设备认证证书）
│  (厂商签发)     │  由 PAI 证书签发，包含设备唯一信息
└─────────────────┘
```

### 证书链验证流程

1. **CD 验证**：验证 CD 证书的自签名特性，确认设备已声明其认证状态
2. **PAI 验证**：使用 CSA 根证书验证 PAI 证书的签名，确认中间证书合法
3. **DAC 验证**：使用 PAI 证书验证 DAC 证书的签名，确认设备身份

这种分层结构实现了：
- **信任委托**：通过中间证书实现灵活的证书管理
- **撤销控制**：可单独吊销 PAI 或 DAC 而不影响其他设备
- **审计追溯**：可追溯设备的完整认证路径

## Matter 配网流程

设备配网是将 Matter 设备加入到 Matter 网络的过程，包含以下阶段：

### 1. Discovery（设备发现）

设备上电后通过 BLE 广播或已建立的 IP 网络进行服务发现：

- 设备发送包含 Matter Service Data 的 BLE 广播
- 或通过 DNS-SD 在局域网中宣告 Matter 设备服务
- Commissioner（配网者）获取设备的 Discriminator 和 Vendor ID

### 2. BLE/PiV 建立连接

- Commissioner 根据发现的设备信息建立 BLE 连接
- 或通过 IP 网络建立安全通道
- 交换协议版本和能力信息

### 3. SPAKE2+ 认证

这是设备配对的核心安全环节：

```
Commissioner                    Device
     │                             │
     │<---- 设备确认 (PAI, DAC) ----│
     │                             │
     │  验证设备证书链               │
     │                             │
     │---- Commissioner身份证明 --->│
     │                             │
     │  SPAKE2+ 验证器比较          │
     │                             │
     │<---- 认证成功/失败 ----------│
```

- 设备将 DAC 和 PAI 证书发送给 Commissioner
- Commissioner 验证证书链的完整性
- 使用 Passcode、Salt 和 Iterations 执行 SPAKE2+ 协议
- 验证通过后建立共享密钥

### 4. Network Commissioning（网络配置）

认证完成后，Commissioner 指导设备完成网络配置：

- 设备扫描并加入 Wi-Fi 或 Thread 网络
- 设备获取必要的网络配置参数
- 设备在 Matter 网络中完成注册
- 建立安全的 Matter 数据通道

## 代码示例

### MFD 初始化与数据读取

```c
#include "bl_mfd.h"
#include <stdio.h>

void matter_device_init(void)
{
    uint32_t size;
    uint8_t *dac_cert;
    uint8_t *dac_key;
    uint8_t vendor_id[2];
    uint8_t product_id[2];
    uint8_t passcode[16];
    uint8_t discriminator[2];
    uint8_t spake2_it[4];
    uint8_t spake2_salt[32];
    uint8_t spake2_verifier[256];
    
    /* 初始化 MFD 模块 */
    if (!mfd_init()) {
        printf("MFD init failed\r\n");
        return;
    }
    printf("MFD init success\r\n");
    
    /* 获取 DAC 证书 */
    dac_cert = mfd_getDacCertPtr(&size);
    if (dac_cert) {
        printf("DAC Cert size: %u\r\n", size);
    }
    
    /* 获取 DAC 私钥 */
    dac_key = mfd_getDacPrivateKeyPtr(&size);
    if (dac_key) {
        printf("DAC Private Key size: %u\r\n", size);
    }
    
    /* 获取厂商和产品信息 */
    mfd_getVendorId(vendor_id, sizeof(vendor_id));
    mfd_getProductId(product_id, sizeof(product_id));
    printf("Vendor ID: 0x%04x, Product ID: 0x%04x\r\n",
           (vendor_id[0] << 8) | vendor_id[1],
           (product_id[0] << 8) | product_id[1]);
    
    /* 获取配网参数 */
    mfd_getPasscode(passcode, sizeof(passcode));
    mfd_getDiscriminator(discriminator, sizeof(discriminator));
    printf("Passcode: %s, Discriminator: %u\r\n", passcode,
           (discriminator[0] << 8) | discriminator[1]);
    
    /* 获取 SPAKE2+ 认证参数 */
    mfd_getSapke2It(spake2_it, sizeof(spake2_it));
    mfd_getSapke2Salt(spake2_salt, sizeof(spake2_salt));
    mfd_getSapke2Verifier(spake2_verifier, sizeof(spake2_verifier));
}
```

### 证书链验证

```c
#include "bl_mfd.h"
#include <stdio.h>

int verify_device_certificate_chain(void)
{
    uint8_t dac_cert[2048];
    uint8_t pai_cert[2048];
    uint8_t cd[2048];
    int ret;
    
    /* 获取三层证书 */
    ret = mfd_getDacCert(dac_cert, sizeof(dac_cert));
    if (ret != 0) {
        printf("Failed to get DAC cert\r\n");
        return -1;
    }
    
    ret = mfd_getPaiCert(pai_cert, sizeof(pai_cert));
    if (ret != 0) {
        printf("Failed to get PAI cert\r\n");
        return -1;
    }
    
    ret = mfd_getCd(cd, sizeof(cd));
    if (ret != 0) {
        printf("Failed to get CD\r\n");
        return -1;
    }
    
    /* 执行证书链验证 */
    /* 实际验证需要调用 Matter SDK 的证书验证接口 */
    printf("Certificate chain retrieved successfully\r\n");
    printf("DAC -> PAI -> CD\r\n");
    
    return 0;
}
```

## 数据存储结构

MFD 数据存储在 Flash 的特定分区，采用 TLV（Type-Length-Value）格式组织：

| Element ID | 描述 | 数据类型 |
|------------|------|----------|
| 0x0001 | DAC 证书 | Certificate |
| 0x0002 | DAC 私钥 | Private Key (加密) |
| 0x0003 | PAI 证书 | Certificate |
| 0x0004 | CD | Certificate |
| 0x0005 | Passcode | UTF-8 String |
| 0x0006 | Discriminator | Integer |
| 0x0007 | Vendor ID | Integer |
| 0x0008 | Product ID | Integer |
| 0x0009 | Vendor Name | String |
| 0x000A | Product Name | String |
| 0x000B | SPAKE2+ Iterations | Integer |
| 0x000C | SPAKE2+ Salt | Binary |
| 0x000D | SPAKE2+ Verifier | Binary |
| 0x000E | Rotating Device ID Unique ID | Binary |

可通过 `mfd_getElementById()` 接口访问任意 Element ID 对应的数据。

## 安全考虑

1. **私钥保护**：DAC 私钥在 Flash 中加密存储，需要 Boot2 或安全引擎配合解密
2. **数据完整性**：工厂数据区应具有完整性保护机制，防止数据被篡改
3. **访问控制**：敏感数据接口应有适当的访问权限控制
4. **安全擦除**：设备退网时应安全擦除认证数据

## 参考

- [Matter Protocol Specification](https://csa-iot.org/)
- BL616/BL618 Bouffalo SDK MFD Component
  - `components/wireless/matter/mfd/include/bl_mfd.h`
  - `components/wireless/matter/mfd/bl_mfd_bl616.h`
