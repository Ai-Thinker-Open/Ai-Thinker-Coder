# BL616/618 Thread 技术文档

## 概述

Thread 是一种基于 IEEE 802.15.4 的低功耗无线 mesh 网络协议，专为智能家居和物联网设备设计。BL616/618 通过集成 OpenThread 协议栈提供完整的 Thread 网络功能支持，实现 IPv6 可寻址的低功耗 mesh 组网。

Thread 协议栈工作在 2.4GHz 频段，最大支持 32 个设备的 mesh 网络，理论传输速率可达 250kbps。与传统点对点协议不同，Thread 支持设备自组成网络、自愈合和大规模设备互联。

## Thread 技术特点

### 6LoWPAN 压缩

Thread 基于 6LoWPAN（IPv6 over Low-Power Wireless Personal Area Networks）协议栈，通过帧头压缩和分片重组技术，在 802.15.4 的有限帧长度（127 字节）内高效传输 IPv6 数据包。这种设计使 Thread 设备可以直接与互联网通信，无需额外的网关协议转换。

### 原生 IPv6 支持

Thread 设备获得完整的 IPv6 地址，支持原生 TCP/UDP 传输层协议。这意味着开发者可以直接使用标准 socket API 开发物联网应用，无需关心底层无线通信细节。Thread 网络支持 Thread 1.1/1.2/1.3 版本，兼容 Matter 协议栈。

### Mesh 网络拓扑

Thread 采用 mesh 网络拓扑结构，所有路由器设备可以相互转发数据，使网络覆盖范围可以随设备数量线性扩展。数据包通过路由算法自动选择最优路径，网络中存在单点故障时流量自动切换到备用路径。

### 自愈合能力

当网络中的路由器设备离开或失效时，剩余路由器自动重新选主，新的路由器从 EndDevice 升级生成。整个过程无需外部干预，网络自动恢复连通性。这种自愈合机制使 Thread 网络具有极高的可靠性。

## Thread 与 BLE Mesh 对比

| 特性 | Thread | BLE Mesh |
|------|--------|----------|
| 物理层 | IEEE 802.15.4 | Bluetooth LE 4.x/5.x |
| 频段 | 2.4GHz | 2.4GHz |
| 传输速率 | 250kbps | 1-2Mbps |
| 最大设备数 | 32 路由器 | 4096 节点 |
| IPv6 支持 | 原生 | 需转换 |
| 功耗 | 低 | 极低 |
| 典型应用 | 骨干网络、中控 | 低功耗传感器 |

Thread 适合作为智能家居骨干网络，连接需要持续通信的中控设备、网关和智能电器。BLE Mesh 则适合低功耗传感器节点，如门窗传感器、温湿度计等。

## 核心概念与设备角色

### Leader（领导者）

Leader 是 Thread 网络中的核心路由器，负责管理整个网络域。它维护网络配置信息、协调设备入网、分配网络地址。Leader 可以由任何路由器自动选举产生，当原有 Leader 失效时网络自动重新选举。

### Router（路由器）

Router 负责转发网络流量，维护路由表，支持子设备入网。Router 具有稳定的电力供应，是网络的骨干节点。Router 可以升级为 Leader，也可以降级为 REED。

### EndDevice（终端设备）

EndDevice 是 Leaf 节点，不参与路由转发，只能与父设备通信。EndDevice 可以进入深度睡眠以节省功耗，适合电池供电设备。EndDevice 无法成为 Router。

### REED（路由器 eligible End Device）

REED 是具有路由器资格的终端设备。当网络需要更多路由器时，REED 可以升级为 Router。BL616/618 默认以 REED 角色加入网络，根据网络拓扑需求动态调整角色。

## OpenThread 平台接口

BL616/618 的 Thread 功能通过 OpenThread 协议栈实现，平台层接口定义如下：

### 初始化函数

```c
// 初始化 OpenThread 协议栈
void otrStackInit(void);

// 启动 OpenThread 任务
void otrStart(void);

// 获取 OpenThread 实例
otInstance *otrGetInstance(void);

// 用户初始化回调（在 OpenThread 任务中调用）
void otrInitUser(otInstance *instance);
```

### 无线电接口

```c
// 初始化 802.15.4 无线电
void ot_radioInit(void);

// 无线电事件处理
void ot_radioTask(ot_system_event_t trxEvent);
```

### UART CLI 接口

```c
// 初始化 OpenThread CLI（用于调试）
void otAppCliInit(otInstance *aInstance);

// 初始化 NCP 模式
void otAppNcpInit(otInstance *aInstance);
```

### 系统事件

```c
typedef enum _ot_system_event {
    OT_SYSTEM_EVENT_NONE                = 0,
    OT_SYSTEM_EVENT_OT_TASKLET          = 0x00000001,  // 任务事件
    OT_SYSTEM_EVENT_ALARM_MS_EXPIRED    = 0x00000002,  // 毫秒定时器
    OT_SYSTEM_EVENT_ALARM_US_EXPIRED    = 0x00000004,  // 微秒定时器
    OT_SYSTEM_EVENT_RADIO_TX_DONE       = 0x00000100,  // 发送完成
    OT_SYSTEM_EVENT_RADIO_RX_DONE       = 0x00002000,  // 接收完成
    OT_SYSTEM_EVENT_POLL                = 0x00010000,  // 数据轮询
    // ... 更多事件
} ot_system_event_t;
```

## 代码示例

### 基本初始化流程

```c
#include "openthread_port.h"
#include <openthread/thread.h>
#include <openthread/instance.h>

// 用户初始化回调实现
void otrInitUser(otInstance *instance) {
    // 设置网络名称
    otThreadSetNetworkName(instance, "MyThreadNet");

    // 设置 PAN ID
    otLinkSetPanId(instance, 0x1234);

    // 设置扩展 PAN ID
    uint8_t extPanId[] = {0xdead, 0xbeef, 0xca, 0xfe, 0xba, 0xbe, 0xfa, 0xce};
    otThreadSetExtendedPanId(instance, extPanId);

    // 启动 Thread 协议栈
    otIp6SetEnabled(instance, true);
    otThreadSetEnabled(instance, true);

    printf("Thread network started\r\n");
}

void app_main(void) {
    // 初始化无线电
    ot_radioInit();

    // 初始化 OpenThread 栈
    otrStackInit();

    // 启动 OpenThread 任务
    otrStart();
}
```

### 设备角色查询

```c
void printDeviceRole(otInstance *instance) {
    otDeviceRole role = otThreadGetDeviceRole(instance);

    switch (role) {
        case OT_DEVICE_ROLE_DISABLED:
            printf("Role: Disabled\r\n");
            break;
        case OT_DEVICE_ROLE_DETACHED:
            printf("Role: Detached\r\n");
            break;
        case OT_DEVICE_ROLE_CHILD:
            printf("Role: EndDevice/REED\r\n");
            break;
        case OT_DEVICE_ROLE_ROUTER:
            printf("Role: Router\r\n");
            break;
        case OT_DEVICE_ROLE_LEADER:
            printf("Role: Leader\r\n");
            break;
    }
}
```

### 网络信息查询

```c
void printNetworkInfo(otInstance *instance) {
    if (!otThreadGetDeviceRole(instance)) {
        return;
    }

    // 获取 RLOC16（路由定位符）
    uint16_t rloc16 = otThreadGetRloc16(instance);
    printf("RLOC16: 0x%04x\r\n", rloc16);

    // 获取路由器数量
    uint8_t routerCount = 0;
    otThreadGetRouterCount(instance, &routerCount);
    printf("Router Count: %d\r\n", routerCount);

    // 获取 Leader 路由器 ID
    uint8_t leaderRouterId = otThreadGetLeaderRouterId(instance);
    printf("Leader Router ID: %d\r\n", leaderRouterId);

    // 获取网络名称
    char name[32];
    otThreadGetNetworkName(instance, name, sizeof(name));
    printf("Network Name: %s\r\n", name);
}
```

## 配置参数

### 任务配置

```c
// OpenThread 任务栈大小（默认 1024）
#ifndef OT_TASK_SIZE
#define OT_TASK_SIZE 1024
#endif

// OpenThread 任务优先级（默认 20）
#ifndef OT_TASK_PRORITY
#define OT_TASK_PRORITY 20
#endif
```

### 无线电配置

```c
// 接收帧缓冲区数量（默认 8）
#ifndef OTRADIO_RX_FRAME_BUFFER_NUM
#define OTRADIO_RX_FRAME_BUFFER_NUM 8
#endif
```

### UART 配置

```c
// UART 接收缓冲区大小（默认 256）
#ifndef OT_UART_RX_BUFFSIZE
#define OT_UART_RX_BUFFSIZE 256
#endif
```

## 事件驱动编程

OpenThread 使用事件驱动模型，主要事件类型：

```c
// 在主循环中处理事件
void processThreadEvents(void) {
    ot_system_event_t event = otrGetEvents();

    if (event & OT_SYSTEM_EVENT_OT_TASKLET) {
        // 处理 OpenThread 任务事件
        otTaskletsProcess(otrGetInstance());
    }

    if (event & OT_SYSTEM_EVENT_RADIO_RX_DONE) {
        // 处理接收完成事件
    }

    if (event & OT_SYSTEM_EVENT_RADIO_TX_DONE) {
        // 处理发送完成事件
    }
}
```

## Thread 网络安全

Thread 使用 802.15.4 安全帧加密，支持 AES-128 加密算法。网络安全密钥通过网络配网过程分发，设备入网前需要通过预共享密钥或证书认证。

关键安全特性：
- 帧级加密（AES-CCM-128）
- 设备身份验证（KEK/MLEK 密钥）
- 定期密钥轮换
- 安全计数器防重放攻击

## 参考

- [OpenThread 官方文档](https://openthread.io/)
- [Thread Group 规范](https://www.threadgroup.org/)
- [BL618 OpenThread 源码](../bouffalo_sdk/components/wireless/thread/openthread_port/)
- [Matter 协议集成](matter.md)
