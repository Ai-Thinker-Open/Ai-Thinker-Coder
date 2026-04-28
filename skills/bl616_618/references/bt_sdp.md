# Bluetooth SDP (Service Discovery Protocol) 技术文档

## 1. 概述

SDP（Service Discovery Protocol，服务发现协议）是蓝牙协议栈中用于查询远程设备支持服务的核心协议。SDP 基于 L2CAP（Logical Link Control and Adaptation Protocol）传输层运行，提供一种标准化的机制，使蓝牙设备能够发现彼此提供的服务及其属性。

当两个蓝牙设备建立连接后，SDP 允许客户端设备向服务器设备发送查询请求，以获取对方支持的服务列表、服务属性以及服务特征信息。通过 SDP，应用程序可以确定远程设备支持哪些蓝牙配置文件（如耳机、串口端口、音频源等），从而决定如何与该设备进行交互。

### 1.1 SDP 协议架构

SDP 采用客户端-服务器模型工作：

- **SDP 服务器**：维护一个服务记录（Service Record）数据库，每个服务记录包含服务类型、服务属性等信息
- **SDP 客户端**：向服务器发送 SDP 查询请求，发现可用服务

SDP 本身不建立通信会话，只是提供一种服务发现机制。实际的数据传输依赖于查询到的服务对应的协议（如 RFCOMM、AVDTP 等）。

## 2. 服务类别标识符

蓝牙assigned numbers定义了标准化的服务类别标识符（Service Class Identifier），用于唯一标识各类蓝牙服务。以下是常见的核心服务类别宏定义：

### 2.1 基础服务类别

| 宏定义 | 值 | 说明 |
|--------|-----|------|
| `BT_SDP_SDP_SERVER_SVCLASS` | 0x1000 | SDP 服务器本身 |
| `BT_SDP_BROWSE_GRP_DESC_SVCLASS` | 0x1001 | 浏览组描述服务 |
| `BT_SDP_PUBLIC_BROWSE_GROUP` | 0x1002 | 公共浏览组 |
| `BT_SDP_GENERIC_ACCESS_SVCLASS` | 0x1800 | 通用访问服务 |
| `BT_SDP_GENERIC_ATTRIB_SVCLASS` | 0x1801 | 通用属性服务 |

### 2.2 常用音频/媒体服务类别

| 宏定义 | 值 | 说明 |
|--------|-----|------|
| `BT_SDP_SERIAL_PORT_SVCLASS` | 0x1101 | 串口端口服务（SPP） |
| `BT_SDP_HEADSET_SVCLASS` | 0x1108 | 耳机服务（HSP） |
| `BT_SDP_AUDIO_SOURCE_SVCLASS` | 0x110a | 音频源服务（A2DP Source） |
| `BT_SDP_AUDIO_SINK_SVCLASS` | 0x110b | 音频接收器服务（A2DP Sink） |
| `BT_SDP_ADVANCED_AUDIO_SVCLASS` | 0x110d | 高级音频服务（A2DP） |
| `BT_SDP_AV_REMOTE_SVCLASS` | 0x110e | AV 远程控制服务（AVRCP） |
| `BT_SDP_AV_REMOTE_TARGET_SVCLASS` | 0x110c | AV 远程目标服务 |
| `BT_SDP_AV_REMOTE_CONTROLLER_SVCLASS` | 0x110f | AV 远程控制器服务 |
| `BT_SDP_VIDEO_SOURCE_SVCLASS` | 0x1303 | 视频源服务 |
| `BT_SDP_VIDEO_SINK_SVCLASS` | 0x1304 | 视频接收器服务 |

### 2.3 其他常用服务类别

| 宏定义 | 值 | 说明 |
|--------|-----|------|
| `BT_SDP_HANDSFREE_SVCLASS` | 0x111e | 免提服务（HFP） |
| `BT_SDP_HANDSFREE_AGW_SVCLASS` | 0x111f | 免提音频网关服务 |
| `BT_SDP_HID_SVCLASS` | 0x1124 | 人机接口设备服务（HID） |
| `BT_SDP_PBAP_PCE_SVCLASS` | 0x112e | 电话簿访问客户端（PBAP） |
| `BT_SDP_PBAP_PSE_SVCLASS` | 0x112f | 电话簿访问服务器（PBAP） |
| `BT_SDP_MAP_MCE_SVCLASS` | 0x1133 | 消息访问客户端（MAP） |
| `BT_SDP_MAP_MSE_SVCLASS` | 0x1132 | 消息访问服务器（MAP） |
| `BT_SDP_PNP_INFO_SVCLASS` | 0x1200 | 即插即用信息服务 |

完整的蓝牙服务类别定义可参考 Bluetooth Assigned Numbers 规范。

## 3. SDP 数据结构

### 3.1 数据元素结构

```c
struct bt_sdp_data_elem {
    uint8_t        type;       // 数据类型描述符
    uint32_t       data_size;  // 数据实际大小
    uint32_t       total_size; // 元素总大小
    const void    *data;       // 指向数据的指针
};
```

数据类型描述符（type字段）由数据类型（高5位）和大小描述符（低3位）组成，主要包括：

- `BT_SDP_UINT8` / `BT_SDP_UINT16` / `BT_SDP_UINT32` / `BT_SDP_UINT64` / `BT_SDP_UINT128`：无符号整数
- `BT_SDP_INT8` / `BT_SDP_INT16` / `BT_SDP_INT32`：有符号整数
- `BT_SDP_UUID16` / `BT_SDP_UUID32` / `BT_SDP_UUID128`：UUID标识符
- `BT_SDP_TEXT_STR8` / `BT_SDP_TEXT_STR16`：文本字符串
- `BT_SDP_SEQ8` / `BT_SDP_SEQ16`：数据序列
- `BT_SDP_BOOL`：布尔类型
- `BT_SDP_URL_STR8`：URL字符串

### 3.2 SDP 属性结构

```c
struct bt_sdp_attribute {
    uint16_t                id;    // 属性ID
    struct bt_sdp_data_elem val;   // 属性数据值
};
```

常用属性ID定义：

| 属性ID | 说明 |
|--------|------|
| `BT_SDP_ATTR_RECORD_HANDLE` | 服务记录句柄 |
| `BT_SDP_ATTR_SVCLASS_ID_LIST` | 服务类别ID列表 |
| `BT_SDP_ATTR_RECORD_STATE` | 记录状态 |
| `BT_SDP_ATTR_SERVICE_ID` | 服务ID |
| `BT_SDP_ATTR_PROTO_DESC_LIST` | 协议描述符列表 |
| `BT_SDP_ATTR_BROWSE_GRP_LIST` | 浏览组列表 |
| `BT_SDP_ATTR_PROFILE_DESC_LIST` | 配置描述符列表 |
| `BT_SDP_ATTR_SUPPORTED_FEATURES` | 支持的特性 |
| `BT_SDP_ATTR_SVCNAME_PRIMARY` | 主服务名称 |
| `BT_SDP_ATTR_SVCDESC_PRIMARY` | 主服务描述 |

### 3.3 SDP 服务记录结构

```c
struct bt_sdp_record {
    uint32_t                    handle;      // 服务记录句柄
    struct bt_sdp_attribute    *attrs;       // 属性数组指针
    size_t                      attr_count;  // 属性数量
    uint8_t                     index;       // 记录索引
    struct bt_sdp_record       *next;        // 指向下一条记录的指针
};
```

### 3.4 SDP 客户端查询结果结构

```c
struct bt_sdp_client_result {
    struct net_buf        *resp_buf;           // 包含未解析的SDP记录结果
    bool                   next_record_hint;  // 指示是否有更多结果
    const struct bt_uuid  *uuid;              // 查询的UUID对象
};
```

### 3.5 SDP 发现参数结构

```c
struct bt_sdp_discover_params {
    sys_snode_t            _node;
    const struct bt_uuid  *uuid;              // 要发现的服务UUID
    bt_sdp_discover_func_t  func;             // 发现结果回调函数
    struct net_buf_pool    *pool;             // 用于SDP查询结果的内存池
};
```

## 4. UUID 与连接关联

蓝牙 UUID 用于唯一标识各类服务和配置文件。SDP 协议使用 `bt_uuid_t` 结构体来表示 UUID，并通过 `bt_conn_t` 结构体表示的连接与远程设备进行 SDP 通信。

### 4.1 UUID 结构体类型

```c
struct bt_uuid {
    u8_t type;  // UUID类型：BT_UUID_TYPE_16、BT_UUID_TYPE_32或BT_UUID_TYPE_128
};

struct bt_uuid_16 {
    struct bt_uuid uuid;
    u16_t val;   // 16位UUID值
};

struct bt_uuid_128 {
    struct bt_uuid uuid;
    u8_t val[16]; // 128位UUID值
};
```

### 4.2 UUID 宏定义

```c
// 初始化16位UUID
#define BT_UUID_INIT_16(value) { .uuid = { BT_UUID_TYPE_16 }, .val = (value) }

// 声明16位UUID
#define BT_UUID_DECLARE_16(value) ((struct bt_uuid *)((struct bt_uuid_16[]) {BT_UUID_INIT_16(value)}))

// 从bt_uuid指针获取bt_uuid_16结构体
#define BT_UUID_16(__u) CONTAINER_OF(__u, struct bt_uuid_16, uuid)
```

### 4.3 UUID 与连接关联

SDP 查询通过 `bt_conn_t` 连接对象与远程设备关联。当调用 `bt_sdp_discover()` 时，需要传入有效的 `bt_conn` 指针来指定要查询的远程设备。查询结果通过回调函数返回，开发者可以在回调中访问 `bt_conn` 信息以识别是哪个设备的查询结果。

## 5. SDP 客户端 API

### 5.1 bt_sdp_discover() - 启动服务发现

```c
int bt_sdp_discover(struct bt_conn *conn,
                    const struct bt_sdp_discover_params *params);
```

**功能描述**：启动对远程设备的 SDP 服务发现会话。

**参数说明**：
- `conn`：标识到远程设备连接的对象指针
- `params`：SDP 发现参数，包含要查询的 UUID、回调函数和结果缓冲区池

**返回值**：成功返回0，失败返回负值错误码。

**使用说明**：
- 该函数执行异步 SDP 查询，发现完成时调用用户提供的回调函数
- 如果当前有 SDP 事务在进行，新请求会被排队等待处理
- 回调函数的返回值可以控制是否继续获取更多记录

**回调函数类型**：

```c
typedef uint8_t (*bt_sdp_discover_func_t)
        (struct bt_conn *conn, struct bt_sdp_client_result *result);
```

回调返回值：
- `BT_SDP_DISCOVER_UUID_STOP`：停止继续获取更多记录
- `BT_SDP_DISCOVER_UUID_CONTINUE`：继续获取下一个记录

### 5.2 bt_sdp_discover_cancel() - 取消服务发现

```c
int bt_sdp_discover_cancel(struct bt_conn *conn,
                           const struct bt_sdp_discover_params *params);
```

**功能描述**：取消正在等待的 SDP 发现请求。

**参数说明**：
- `conn`：标识到远程设备连接的对象指针
- `params`：要取消的 SDP 发现参数

**返回值**：成功返回0，失败返回负值错误码。

### 5.3 bt_sdp_get_proto_param() - 获取协议参数

```c
int bt_sdp_get_proto_param(const struct net_buf *buf, enum bt_sdp_proto proto,
                           uint16_t *param);
```

**功能描述**：从 SDP 记录中提取特定协议的参数值。

**支持的协议**：
- `BT_SDP_PROTO_RFCOMM = 0x0003`：RFCOMM 协议
- `BT_SDP_PROTO_L2CAP = 0x0100`：L2CAP 协议

**参数说明**：
- `buf`：包含原始 SDP 记录数据的缓冲区
- `proto`：要查询的协议类型
- `param`：成功时填充找到的参数值

**返回值**：成功返回0，失败返回负值错误码。

### 5.4 bt_sdp_get_profile_version() - 获取配置文件版本

```c
int bt_sdp_get_profile_version(const struct net_buf *buf, uint16_t profile,
                               uint16_t *version);
```

**功能描述**：从 SDP 记录中提取远程设备的配置文件版本号。

**参数说明**：
- `buf`：原始 SDP 记录数据缓冲区
- `profile`：配置文件家族标识符
- `version`：成功时填充找到的版本号

**返回值**：成功返回0，失败返回负值错误码。

### 5.5 bt_sdp_get_features() - 获取支持的特性

```c
int bt_sdp_get_features(const struct net_buf *buf, uint16_t *features);
```

**功能描述**：从 SDP 记录中获取远程设备支持的特性列表。

**参数说明**：
- `buf`：包含原始 SDP 记录数据的缓冲区
- `features`：成功时填充 SupportedFeature 属性掩码

**返回值**：成功返回0，记录中无特性信息或解析失败返回负值错误码。

## 6. SDP 服务器端 API

### 6.1 bt_sdp_register_service() - 注册服务记录

```c
int bt_sdp_register_service(struct bt_sdp_record *service);
```

**功能描述**：向本地 SDP 数据库注册一个服务记录，使远程设备能够通过 SDP 查询发现该服务。

**参数说明**：
- `service`：使用 `BT_SDP_RECORD` 宏声明的服务记录结构

**返回值**：成功返回0，失败返回负值错误码。

### 6.2 bt_sdp_init() - 初始化 SDP

```c
void bt_sdp_init(void);
```

**功能描述**：初始化 SDP 子系统，调用后才可以注册服务记录或执行客户端查询。

## 7. SDP 辅助宏

### 7.1 服务记录声明宏

```c
// 声明一个新的服务记录，包含默认属性
#define BT_SDP_NEW_SERVICE { ... }

// 声明一个完整的服务记录
#define BT_SDP_RECORD(_attrs) { .attrs = _attrs, .attr_count = ARRAY_SIZE((_attrs)) }
```

### 7.2 属性声明宏

```c
// 声明列表类型属性
#define BT_SDP_LIST(_att_id, _type_size, _data_elem_seq)

// 声明服务ID属性
#define BT_SDP_SERVICE_ID(_uuid)

// 声明服务名称属性
#define BT_SDP_SERVICE_NAME(_name)

// 声明支持的特性属性
#define BT_SDP_SUPPORTED_FEATURES(_features)
```

### 7.3 数据元素辅助宏

```c
// 声明8位数组
#define BT_SDP_ARRAY_8(...) ((uint8_t[]) {__VA_ARGS__})

// 声明16位数组
#define BT_SDP_ARRAY_16(...) ((uint16_t[]) {__VA_ARGS__})

// 声明32位数组
#define BT_SDP_ARRAY_32(...) ((uint32_t[]) {__VA_ARGS__})

// 声明固定大小数据元素
#define BT_SDP_TYPE_SIZE(_type)

// 声明可变大小数据元素
#define BT_SDP_TYPE_SIZE_VAR(_type, _size)

// 声明数据元素列表
#define BT_SDP_DATA_ELEM_LIST(...) ((struct bt_sdp_data_elem[]) {__VA_ARGS__})
```

## 8. 代码示例

### 8.1 服务发现示例

以下示例展示如何发现远程设备支持的服务：

```c
#include "bluetooth/sdp.h"
#include "bluetooth/conn.h"
#include "bluetooth/bt_uuid.h"
#include <net/buf.h>

/* 用于存储发现结果的缓冲区池 */
static struct net_buf_pool *sdp_result_pool;

/* SDP 发现回调函数 */
static uint8_t sdp_discover_cb(struct bt_conn *conn,
                               struct bt_sdp_client_result *result)
{
    uint16_t features;
    int err;

    if (!result || !result->resp_buf) {
        printk("SDP 发现完成，无更多结果\n");
        return BT_SDP_DISCOVER_UUID_STOP;
    }

    printk("发现服务 UUID: 0x%04x\n",
           result->uuid ? ((struct bt_uuid_16 *)result->uuid)->val : 0);

    /* 尝试获取该服务支持的功能特性 */
    err = bt_sdp_get_features(result->resp_buf, &features);
    if (err == 0) {
        printk("  支持的特性: 0x%04x\n", features);
    }

    /* 根据 next_record_hint 决定是否继续 */
    if (result->next_record_hint) {
        return BT_SDP_DISCOVER_UUID_CONTINUE;
    }

    return BT_SDP_DISCOVER_UUID_STOP;
}

/* 启动 SDP 服务发现 */
int discover_remote_services(struct bt_conn *conn)
{
    struct bt_sdp_discover_params discover_params;
    static const struct bt_uuid_16 sdp_uuid = BT_UUID_INIT_16(
        BT_SDP_PUBLIC_BROWSE_GROUP);

    /* 初始化发现参数 */
    discover_params.uuid = &sdp_uuid.uuid;
    discover_params.func = sdp_discover_cb;
    discover_params.pool = sdp_result_pool;

    printk("开始发现远程设备服务...\n");

    return bt_sdp_discover(conn, &discover_params);
}
```

### 8.2 查询特定服务示例

以下示例展示如何查询远程设备的串口端口服务（SPP）：

```c
#include "bluetooth/sdp.h"
#include "bluetooth/conn.h"
#include "bluetooth/bt_uuid.h"

/* SPP 服务查询回调 */
static uint8_t spp_discover_cb(struct bt_conn *conn,
                                struct bt_sdp_client_result *result)
{
    uint16_t rfcomm_channel = 0;
    int err;

    if (!result || !result->resp_buf) {
        return BT_SDP_DISCOVER_UUID_STOP;
    }

    /* 从 SDP 记录中获取 RFCOMM 通道号 */
    err = bt_sdp_get_proto_param(result->resp_buf, BT_SDP_PROTO_RFCOMM,
                                 &rfcomm_channel);
    if (err == 0) {
        printk("SPP 服务 - RFCOMM 通道: %d\n", rfcomm_channel);
    } else {
        printk("SPP 服务 - 无法获取 RFCOMM 通道\n");
    }

    return BT_SDP_DISCOVER_UUID_STOP;
}

/* 查询远程设备的 SPP 服务 */
int discover_spp_service(struct bt_conn *conn)
{
    struct bt_sdp_discover_params params;
    static const struct bt_uuid_16 spp_uuid = BT_UUID_INIT_16(
        BT_SDP_SERIAL_PORT_SVCLASS);

    params.uuid = &spp_uuid.uuid;
    params.func = spp_discover_cb;
    params.pool = sdp_result_pool;

    printk("查询 SPP 服务...\n");
    return bt_sdp_discover(conn, &params);
}
```

### 8.3 注册本地 SDP 服务示例

以下示例展示如何在本地注册一个自定义蓝牙服务：

```c
#include "bluetooth/sdp.h"

/* 定义服务属性 */
static const struct bt_sdp_attribute my_service_attrs[] = {
    /* 基础服务声明 - 必须 */
    BT_SDP_NEW_SERVICE,

    /* 服务类别 ID */
    BT_SDP_LIST(BT_SDP_ATTR_SVCLASS_ID_LIST,
                BT_SDP_TYPE_SIZE(BT_SDP_SEQ8),
                BT_SDP_DATA_ELEM_LIST({
                    BT_SDP_TYPE_SIZE(BT_SDP_UUID16),
                    BT_SDP_ARRAY_16(BT_SDP_SERIAL_PORT_SVCLASS)
                })),

    /* 协议描述符列表 */
    BT_SDP_LIST(BT_SDP_ATTR_PROTO_DESC_LIST,
                BT_SDP_TYPE_SIZE(BT_SDP_SEQ8),
                BT_SDP_DATA_ELEM_LIST({
                    BT_SDP_TYPE_SIZE(BT_SDP_SEQ8),
                    BT_SDP_DATA_ELEM_LIST({
                        BT_SDP_TYPE_SIZE(BT_SDP_UUID16),
                        BT_SDP_ARRAY_16(BT_SDP_PROTO_L2CAP)
                    }),
                    BT_SDP_TYPE_SIZE(BT_SDP_UINT16),
                    BT_SDP_ARRAY_16(0x0003)  /* RFCOMM */
                })),

    /* 服务名称 */
    BT_SDP_SERVICE_NAME("My Custom Service"),

    /* 支持的特性 */
    BT_SDP_SUPPORTED_FEATURES(0x0001),
};

/* 声明服务记录 */
static const struct bt_sdp_record my_service_record = {
    BT_SDP_RECORD(my_service_attrs)
};

/* 注册服务 */
int register_my_service(void)
{
    int err;

    err = bt_sdp_register_service(&my_service_record);
    if (err < 0) {
        printk("服务注册失败: %d\n", err);
        return err;
    }

    printk("自定义服务注册成功\n");
    return 0;
}
```

### 8.4 遍历服务列表示例

以下示例展示如何遍历浏览组下的所有服务：

```c
#include "bluetooth/sdp.h"
#include "bluetooth/bt_uuid.h"

/* 浏览回调 - 打印所有发现的服务 */
static uint8_t browse_all_cb(struct bt_conn *conn,
                             struct bt_sdp_client_result *result)
{
    struct bt_uuid_16 *uuid;
    char uuid_str[8];

    if (!result || !result->resp_buf) {
        printk("浏览完成\n");
        return BT_SDP_DISCOVER_UUID_STOP;
    }

    /* 从回调结果中获取 UUID */
    if (result->uuid && result->uuid->type == BT_UUID_TYPE_16) {
        uuid = BT_UUID_16(result->uuid);
        snprintf(uuid_str, sizeof(uuid_str), "0x%04x", uuid->val);

        /* 根据 UUID 值识别常见服务 */
        const char *name = "未知服务";
        switch (uuid->val) {
        case 0x1101: name = "串口端口(SPP)"; break;
        case 0x1108: name = "耳机(HSP)"; break;
        case 0x110a: name = "音频源"; break;
        case 0x110d: name = "高级音频(A2DP)"; break;
        case 0x110e: name = "AV远程控制"; break;
        case 0x111e: name = "免提(HFP)"; break;
        }

        printk("发现服务: %s [%s]\n", name, uuid_str);
    }

    return result->next_record_hint ?
           BT_SDP_DISCOVER_UUID_CONTINUE : BT_SDP_DISCOVER_UUID_STOP;
}

/* 浏览所有可用服务 */
int browse_all_services(struct bt_conn *conn)
{
    struct bt_sdp_discover_params params;
    static const struct bt_uuid_16 browse_uuid = BT_UUID_INIT_16(
        BT_SDP_BROWSE_GRP_DESC_SVCLASS);

    params.uuid = &browse_uuid.uuid;
    params.func = browse_all_cb;
    params.pool = sdp_result_pool;

    printk("开始浏览所有服务...\n");
    return bt_sdp_discover(conn, &params);
}
```

## 9. SDP 工作流程

### 9.1 客户端发现服务流程

1. **建立连接**：首先通过 ACL（Asynchronous Connection-Oriented）链路建立蓝牙连接
2. **初始化 SDP**：调用 `bt_sdp_init()` 确保 SDP 子系统就绪
3. **配置发现参数**：设置要查询的 UUID、回调函数和结果缓冲区
4. **发起发现请求**：调用 `bt_sdp_discover()` 发送 SDP 查询
5. **处理结果**：在回调函数中解析返回的 SDP 记录
6. **提取信息**：使用辅助 API（如 `bt_sdp_get_proto_param()`）提取所需属性

### 9.2 服务器端注册服务流程

1. **初始化 SDP**：调用 `bt_sdp_init()` 初始化 SDP 子系统
2. **定义服务属性**：使用辅助宏定义服务记录的属性列表
3. **声明服务记录**：使用 `BT_SDP_RECORD()` 宏创建服务记录结构
4. **注册服务**：调用 `bt_sdp_register_service()` 将服务记录添加到本地 SDP 数据库
5. **等待查询**：远程设备可通过 SDP 查询发现已注册的服务

## 10. 注意事项

### 10.1 连接管理

- SDP 查询必须基于已建立的 `bt_conn` 连接
- 在连接断开后，SDP 查询结果将不再有效
- 建议在连接建立成功后立即进行必要的 SDP 发现

### 10.2 内存管理

- SDP 发现结果存储在 `net_buf` 缓冲区中，需要提供足够大的内存池
- 回调函数返回后，缓冲区可能已被释放，不应继续引用
- 对于大型服务数据库，可能需要多次回调才能获取完整结果

### 10.3 错误处理

- 大部分 SDP API 返回负值表示错误，0 表示成功
- `bt_sdp_get_proto_param()`、`bt_sdp_get_features()` 等辅助函数也返回负值表示解析失败
- 建议始终检查返回值并处理错误情况

### 10.4 UUID 类型匹配

- 确保查询时使用的 UUID 类型与远程设备服务记录中的类型一致
- 16位 UUID 是最常用的形式，对应蓝牙标准服务
- 128位 UUID 用于自定义或供应商特定服务

## 参考

- [Bluetooth Core Specification - SDP Section](https://www.bluetooth.com/specifications/assigned-numbers/service-discovery)
- bouffalo_sdk/components/wireless/bluetooth/btprofile/include/bluetooth/sdp.h
- bouffalo_sdk/components/wireless/bluetooth/blestack/src/include/bluetooth/bt_uuid.h
