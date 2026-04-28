# NetHub 网络过滤框架

## 概述

NetHub 是博流（Bouffalo Lab）BL616/BL618 芯片系列的核心网络框架，提供了完整的推流服务器能力，支持 RTSP、HTTP-FLV、HLS 等多种流媒体协议。该框架的核心功能之一是实现 Wi-Fi 数据包的智能过滤与分发，通过可配置的策略引擎对接收到的数据包进行分类处理，决定数据包应当被丢弃、交付给本地协议栈、转发给 Host 处理器，或者同时执行多种操作。

NetHub 的过滤系统位于 `components/net/nethub/core/` 目录，其中 `nh_filter.h` 定义了完整的过滤数据结构和 API。Wi-Fi 后端相关实现在 `backend/wifi/` 目录下，包含桥接层和通用后端接口。Host 侧的传输支持则通过 `backend/host/sdio/` 等目录下的 SDIO 传输层实现。

## 核心数据类型

### 过滤匹配类型 `nh_filter_match_t`

数据包匹配类型枚举，定义了能够识别的协议和端口范围：

```c
typedef enum {
    NH_FILTER_MATCH_8021X = 0,      // 802.1X 认证帧
    NH_FILTER_MATCH_ARP,            // ARP 协议
    NH_FILTER_MATCH_DHCP4,          // IPv4 DHCP 动态主机配置协议
    NH_FILTER_MATCH_ICMP4,          // IPv4 ICMP 控制报文协议
    NH_FILTER_MATCH_TCP4_DST_PORT_RANGE,  // TCP IPv4 目标端口范围
    NH_FILTER_MATCH_UDP4_DST_PORT_RANGE,  // UDP IPv4 目标端口范围
} nh_filter_match_t;
```

当匹配类型为 `TCP4_DST_PORT_RANGE` 或 `UDP4_DST_PORT_RANGE` 时，需要在规则结构中指定 `port_min` 和 `port_max` 字段来界定端口范围。框架会根据这两个字段判断 incoming 数据包的目标端口是否落在指定范围内。

### 过滤动作类型 `nh_filter_action_t`

过滤动作决定了匹配成功的数据包应当如何处理：

```c
#define NH_FILTER_ACTION_DROP  NETHUB_WIFI_RX_FILTER_DROP   // 丢弃数据包
#define NH_FILTER_ACTION_LOCAL NETHUB_WIFI_RX_FILTER_LOCAL   // 仅交付本地协议栈
#define NH_FILTER_ACTION_HOST  NETHUB_WIFI_RX_FILTER_HOST    // 仅转发给 Host
#define NH_FILTER_ACTION_BOTH  NETHUB_WIFI_RX_FILTER_BOTH    // 同时交付本地和转发 Host
```

`NH_FILTER_ACTION_BOTH` 是一个组合动作，其定义为 `LOCAL | HOST`（按位或），表示数据包既需要在本地协议栈处理（如lwIP），也需要通过 SDIO 等通道传递给 Host 处理器。

### 过滤规则结构 `nh_filter_rule_t`

每条过滤规则由匹配条件、动作和可选参数组成：

```c
typedef struct {
    nh_filter_match_t match;     // 匹配类型
    nh_filter_action_t action;   // 匹配后执行的动作
    uint16_t port_min;           // 端口范围下限（仅对端口范围匹配有效）
    uint16_t port_max;           // 端口范围上限（仅对端口范围匹配有效）
} nh_filter_rule_t;
```

对于非端口相关的匹配类型（如 ARP、DHCP4），`port_min` 和 `port_max` 字段被忽略。

### 过滤策略结构 `nh_filter_policy_t`

策略是规则集与默认动作的集合：

```c
typedef struct {
    const nh_filter_rule_t *rules;      // 指向规则数组的指针
    size_t rule_count;                  // 规则数量
    nh_filter_action_t default_action;  // 未匹配任何规则时的默认动作
} nh_filter_policy_t;
```

当数据包不匹配任何规则时，`default_action` 指定了默认的处理方式。策略在系统初始化时创建，并在数据包处理流程中被引用。

## 通道类型

NetHub 定义了多种数据通道，用于区分不同的数据来源和目的地：

```c
typedef enum {
    NETHUB_CHANNEL_WIFI_STA = 0,   // Wi-Fi Station 模式接收通道
    NETHUB_CHANNEL_WIFI_AP,         // Wi-Fi AP 模式接收通道
    NETHUB_CHANNEL_STACK_STA,       // 本地协议栈 Station 通道
    NETHUB_CHANNEL_STACK_AP,        // 本地协议栈 AP 通道
    NETHUB_CHANNEL_STACK_NAT,       // 本地协议栈 NAT 通道
    NETHUB_CHANNEL_BRIDGE,          // 桥接通道
    NETHUB_CHANNEL_SDIO,            // SDIO 传输通道
    NETHUB_CHANNEL_USB,             // USB 传输通道
    NETHUB_CHANNEL_SPI,             // SPI 传输通道
    NETHUB_CHANNEL_MAX
} nethub_channel_t;
```

Wi-Fi 通道（STA 和 AP）接收到的数据包首先经过过滤策略处理，再根据过滤结果分发到本地协议栈或 Host 端。

## 核心 API

### `nh_filter_wifi_rx()`

Wi-Fi 接收数据包的入口函数，对数据包应用当前注册的过滤策略：

```c
nh_filter_action_t nh_filter_wifi_rx(nethub_channel_t src_channel, const struct pbuf *pkt);
```

**参数说明：**
- `src_channel`：数据包的来源通道（`NETHUB_CHANNEL_WIFI_STA` 或 `NETHUB_CHANNEL_WIFI_AP`）
- `pkt`：指向 lwIP `pbuf` 结构的数据包指针

**返回值：**
- `nh_filter_action_t` 类型的过滤动作，指示数据包应当被如何处理

该函数是 Wi-Fi RX 数据路径上的核心处理点，负责解析数据包协议层并应用过滤规则。

### `nh_filter_apply_policy()`

显式地将过滤策略应用于指定的数据包：

```c
nh_filter_action_t nh_filter_apply_policy(const struct pbuf *pkt, const nh_filter_policy_t *policy);
```

**参数说明：**
- `pkt`：指向待处理数据包的 `pbuf` 指针
- `policy`：指向待应用的过滤策略结构

**返回值：**
- 匹配到的规则对应的动作，或策略的 `default_action`（如无匹配）

此函数允许应用层在运行时动态切换过滤策略，适用于需要根据不同场景调整网络行为的情况。

### `nh_filter_should_deliver_local()`

判断过滤动作是否包含本地交付：

```c
bool nh_filter_should_deliver_local(nh_filter_action_t action);
```

**返回值：**
- `true`：动作包含 `LOCAL` 标志，需要交付本地协议栈
- `false`：动作不包含 `LOCAL` 标志

### `nh_filter_should_deliver_host()`

判断过滤动作是否包含 Host 转发：

```c
bool nh_filter_should_deliver_host(nh_filter_action_t action);
```

**返回值：**
- `true`：动作包含 `HOST` 标志，需要转发给 Host
- `false`：动作不包含 `HOST` 标志

这两个判断函数通常在数据包处理流程中组合使用，以决定数据包的最终路由。例如，动作 `NH_FILTER_ACTION_BOTH` 会同时返回两个函数的 `true` 结果。

### `nh_filter_custom_wifi_rx_is_active()`

查询当前是否注册了自定义 Wi-Fi 过滤回调：

```c
bool nh_filter_custom_wifi_rx_is_active(void);
```

**返回值：**
- `true`：已通过 `nethub_set_wifi_rx_filter()` 注册了自定义过滤回调
- `false`：使用内置默认过滤策略

当自定义过滤回调处于激活状态时，内置的过滤策略会被完全绕过。

### `nethub_set_wifi_rx_filter()`

注册自定义 Wi-Fi 过滤回调函数：

```c
int nethub_set_wifi_rx_filter(nethub_wifi_rx_filter_cb_t filter_cb, void *user_ctx);
```

**回调函数类型定义：**

```c
typedef nethub_wifi_rx_filter_action_t (*nethub_wifi_rx_filter_cb_t)(
    nethub_channel_t src_channel,
    const struct pbuf *pkt,
    void *user_ctx);
```

**重要约束：**
- 此函数必须在 `nethub_bootstrap()` 之前调用
- 回调运行在 Wi-Fi RX 关键路径上，必须满足以下要求：
  - 不得阻塞或进入睡眠
  - 不得释放、保留或修改 `pkt`
  - 不得假设整个帧是连续内存（pbuf 可能是链式结构）
- 传入 `NULL` 可恢复内置策略

## Wi-Fi 后端接口

### `nh_wifi_backend_init()`

初始化 Wi-Fi 后端模块：

```c
int nh_wifi_backend_init(void);
```

**返回值：**
- `0` 或 `NETHUB_OK`：初始化成功
- 负值：初始化失败

### `nh_wifi_backend_tx()`

通过 Wi-Fi 后端发送数据包：

```c
nh_wifi_backend_tx_result_t nh_wifi_backend_tx(struct pbuf *p, bool is_sta);
```

**参数说明：**
- `p`：待发送的数据包
- `is_sta`：`true` 表示通过 Station 接口发送，`false` 表示通过 AP 接口发送

**返回值：**
- `NH_WIFI_BACKEND_TX_OK`：发送成功
- `NH_WIFI_BACKEND_TX_ERR_SEND`：发送失败
- `NH_WIFI_BACKEND_TX_ERR_NETIF_DOWN`：网络接口未就绪

### `nh_wifi_bridge_handle_rx()`

所有 Wi-Fi 后端实现的公共桥接 RX 处理路径：

```c
struct pbuf *nh_wifi_bridge_handle_rx(bool is_sta, struct pbuf *p);
```

该函数封装了数据包从 Wi-Fi 驱动到协议栈或 Host 的完整处理流程，包括过滤策略的应用和分发决策。

### Wi-Fi 端点操作接口

Wi-Fi 端点定义了与具体硬件相关的操作接口：

```c
const struct nhif_ops *nh_wifi_endpoint_get_ops(size_t index);
size_t nh_wifi_endpoint_get_count(void);
```

通过端点索引可获取对应的操作函数集合，支持多种 Wi-Fi 硬件配置。

## SDIO 传输层

SDIO 通道是 BL616/BL618 与 Host 处理器通信的主要路径之一：

```c
extern const struct nhif_ops nhsdio_ops;
extern const nh_ctrlpath_ops_t nhsdio_ctrlpath_ops;
```

这些操作接口由 `transport_sdio.h` 导出，提供了 SDIO 通道的数据传输和控制平面能力。

## 运行时状态

NetHub 提供了运行时状态查询接口：

```c
typedef struct {
    bool initialized;                      // 框架是否已初始化
    bool started;                           // 框架是否已启动
    bool custom_wifi_rx_filter_active;     // 自定义过滤回调是否激活
    const char *profile_name;               // 当前配置文件名
    nethub_channel_t host_channel;          // Host 通道
    nethub_channel_t active_wifi_channel;   // 当前活动的 Wi-Fi 通道
    nethub_statistics_t statistics;         // 流量统计信息
} nethub_runtime_status_t;
```

统计信息包含下载（dnld）和上传（upld）两个方向的数据包计数、丢弃计数、传输成功计数等。

## 代码示例

### 过滤策略初始化

以下示例展示了如何初始化一个基本的过滤策略，允许 DHCP、ARP 和 ICMP 通过，将常见媒体端口的流量转发给 Host，其他流量默认丢弃：

```c
#include "nh_filter.h"

// 定义过滤规则
static const nh_filter_rule_t my_rules[] = {
    // 允许 DHCP 流量交付本地（用于 IP 地址获取）
    {
        .match = NH_FILTER_MATCH_DHCP4,
        .action = NH_FILTER_ACTION_LOCAL,
    },
    // 允许 ARP 流量交付本地（用于 ARP 解析）
    {
        .match = NH_FILTER_MATCH_ARP,
        .action = NH_FILTER_ACTION_LOCAL,
    },
    // 允许 ICMP 流量交付本地（用于 ping 测试）
    {
        .match = NH_FILTER_MATCH_ICMP4,
        .action = NH_FILTER_ACTION_LOCAL,
    },
    // 将 HTTP/RTSP 等媒体端口流量转发给 Host
    {
        .match = NH_FILTER_MATCH_TCP4_DST_PORT_RANGE,
        .action = NH_FILTER_ACTION_HOST,
        .port_min = 80,
        .port_max = 554,  // HTTP 到 RTSP 端口
    },
    // 将 RTMP 推流端口转发给 Host
    {
        .match = NH_FILTER_MATCH_TCP4_DST_PORT_RANGE,
        .action = NH_FILTER_ACTION_HOST,
        .port_min = 1935,
        .port_max = 1935,  // RTMP 默认端口
    },
    // 将 HTTP-FLV 端口转发给 Host
    {
        .match = NH_FILTER_MATCH_TCP4_DST_PORT_RANGE,
        .action = NH_FILTER_ACTION_HOST,
        .port_min = 8080,
        .port_max = 8080,
    },
};

// 创建过滤策略
static const nh_filter_policy_t my_policy = {
    .rules = my_rules,
    .rule_count = sizeof(my_rules) / sizeof(my_rules[0]),
    .default_action = NH_FILTER_ACTION_DROP,  // 默认丢弃未匹配流量
};

// 应用策略的函数
void apply_my_policy(void) {
    nh_filter_action_t action;
    struct pbuf *pkt = NULL;  // 假设这是接收到的数据包

    // 对数据包应用过滤策略
    action = nh_filter_apply_policy(pkt, &my_policy);

    // 根据动作决定路由
    if (nh_filter_should_deliver_local(action)) {
        // 交付给本地 lwIP 协议栈处理
    }
    if (nh_filter_should_deliver_host(action)) {
        // 通过 SDIO 等通道转发给 Host
    }
}
```

### 自定义 Wi-Fi 过滤回调注册

当内置策略无法满足需求时，可注册自定义过滤回调。以下示例展示了一个简单的自定义过滤器，根据数据包的协议类型和来源通道决定处理方式：

```c
#include "nethub_filter.h"
#include "lwip/pbuf.h"

// 自定义用户上下文
typedef struct {
    bool debug_enabled;
    uint32_t dropped_count;
} my_filter_ctx_t;

static my_filter_ctx_t g_my_ctx = {
    .debug_enabled = true,
    .dropped_count = 0,
};

// 自定义过滤回调实现
static nethub_wifi_rx_filter_action_t my_wifi_filter(
    nethub_channel_t src_channel,
    const struct pbuf *pkt,
    void *user_ctx) {

    my_filter_ctx_t *ctx = (my_filter_ctx_t *)user_ctx;

    if (ctx == NULL || pkt == NULL) {
        return NH_FILTER_ACTION_DROP;
    }

    // 简化判断：假设所有 DHCP 流量需要本地处理
    // 实际实现需要解析 pbuf 中的协议头
    nethub_wifi_rx_filter_action_t result = NH_FILTER_ACTION_DROP;

    // 根据来源通道决定行为
    switch (src_channel) {
        case NETHUB_CHANNEL_WIFI_STA:
            // STA 模式下，所有流量都转发给 Host
            result = NH_FILTER_ACTION_HOST;
            break;
        case NETHUB_CHANNEL_WIFI_AP:
            // AP 模式下，仅媒体端口流量转发给 Host
            result = NH_FILTER_ACTION_LOCAL;
            break;
        default:
            result = NH_FILTER_ACTION_DROP;
            break;
    }

    return result;
}

// 初始化自定义过滤
int init_custom_filter(void) {
    int ret;

    // 注册自定义过滤回调（必须在 nethub_bootstrap 之前调用）
    ret = nethub_set_wifi_rx_filter(my_wifi_filter, &g_my_ctx);
    if (ret != 0) {
        return ret;
    }

    // 后续可以检查是否注册成功
    if (nh_filter_custom_wifi_rx_is_active()) {
        // 自定义过滤已激活
    }

    return 0;
}
```

### Wi-Fi 数据包接收与过滤处理

完整的 Wi-Fi RX 处理流程示例：

```c
#include "nh_wifi_backend.h"
#include "nh_filter.h"

// Wi-Fi 接收处理函数
void handle_wifi_rx(bool is_sta, struct pbuf *p) {
    nh_filter_action_t action;

    if (p == NULL) {
        return;
    }

    // 获取来源通道
    nethub_channel_t src_channel = is_sta ?
        NETHUB_CHANNEL_WIFI_STA : NETHUB_CHANNEL_WIFI_AP;

    // 应用过滤策略
    action = nh_filter_wifi_rx(src_channel, p);

    // 根据过滤结果处理数据包
    if (nh_filter_should_deliver_local(action)) {
        // 交付给本地网络协议栈
        // 通过桥接函数交给 lwIP 处理
        struct pbuf *local_pkt = p;
        // nh_wifi_bridge_handle_rx 处理本地交付
    }

    if (nh_filter_should_deliver_host(action)) {
        // 转发给 Host 处理器
        // 通过 SDIO 等通道发送
        nh_wifi_backend_tx_result_t tx_result;
        tx_result = nh_wifi_backend_tx(p, is_sta);

        if (tx_result != NH_WIFI_BACKEND_TX_OK) {
            // 处理发送失败
        }
    }

    // 如果动作为 DROP，数据包在此处自然不被处理，pbuf 将被释放
}
```

## 过滤策略设计建议

### 策略顺序

规则数组中的顺序很重要——框架会按顺序遍历规则并返回第一个匹配的结果。因此，应当将最常用的规则放在数组前面，以提高处理效率。

### 端口范围匹配

对于 TCP/UDP 端口范围匹配，合理规划 `port_min` 和 `port_max` 的值：

- 单个端口：设置 `port_min == port_max`
- 连续端口段：设置范围边界
- 非连续端口：需要多条规则

### 默认动作选择

默认动作的选择取决于具体应用场景：

- `NH_FILTER_ACTION_DROP`：高安全要求场景，仅允许明确指定的流量通过
- `NH_FILTER_ACTION_LOCAL`：需要本地处理所有流量，Host 仅做监控
- `NH_FILTER_ACTION_HOST`：本地仅做转发，大部分处理在 Host 完成
- `NH_FILTER_ACTION_BOTH`：需要本地和 Host 同时处理的场景

### 性能考虑

过滤回调运行在 Wi-Fi RX 关键路径上，应避免：

- 动态内存分配
- 复杂的协议解析（使用快速字段提取）
- 锁竞争和阻塞调用

## 错误码

NetHub 定义了以下错误码：

```c
typedef enum {
    NETHUB_OK = 0,
    NETHUB_ERR_INVALID_PARAM = -1,      // 无效参数
    NETHUB_ERR_NOT_FOUND = -2,          // 未找到请求的资源
    NETHUB_ERR_ALREADY_EXISTS = -3,    // 资源已存在
    NETHUB_ERR_NO_MEMORY = -4,         // 内存不足
    NETHUB_ERR_NOT_INITIALIZED = -5,   // 未初始化
    NETHUB_ERR_INVALID_STATE = -6,     // 无效状态
    NETHUB_ERR_INTERNAL = -7,          // 内部错误
    NETHUB_ERR_NOT_SUPPORTED = -8,     // 不支持的操作
} nethub_status_t;
```

## 参考

- [BL618Claw Bouffalo SDK](https://github.com/bouffalolab/bl_mcu_sdk)
- `components/net/nethub/core/nh_filter.h` — 过滤核心 API 定义
- `components/net/nethub/backend/wifi/nh_wifi_bridge.h` — Wi-Fi 桥接接口
- `components/net/nethub/backend/wifi/nh_wifi_backend.h` — Wi-Fi 后端接口
- `components/net/nethub/backend/host/sdio/transport_sdio.h` — SDIO 传输层定义
- `components/net/nethub/include/nethub_filter.h` — 过滤回调和动作类型
- `components/net/nethub/include/nethub_defs.h` — 通道和状态类型定义
