# BL616/BL618 iperf 网络吞吐量测试组件

## 概述

iperf 是一款广泛使用的网络性能测试工具，主要用于测量 TCP 和 UDP 协议的吞吐量。BL616/BL618 SDK 集成了 iperf 组件，以便开发者能够在嵌入式环境下验证 Wi-Fi 通信性能。通过 iperf 测试，可以获得设备的最大上下行带宽、延迟、抖动等关键网络指标，为 Wi-Fi 应用开发提供性能参考依据。

在 BL616/BL618 平台上，iPerf 组件基于轻量级 TCP/IP 协议栈 lwIP 实现，支持独立的任务运行模式。组件提供了简洁的 API 接口，开发者只需配置必要的参数即可快速启动测试。测试结果可通过串口日志或回调函数获取，便于集成到自动化测试系统中。

## 头文件

```c
#include "iperf.h"
```

## 关键常量定义

### IP 地址类型

```c
#define IPERF_IP_TYPE_IPV4          0
#define IPERF_IP_TYPE_IPV6          1
```

iPerf 支持 IPv4 和 IPv6 两种地址协议族。IPv4 使用 32 位地址，IPv6 使用 128 位地址。默认使用 IPv4，IPv6 主要用于下一代网络环境测试。

### 传输协议类型

```c
#define IPERF_TRANS_TYPE_TCP        0
#define IPERF_TRANS_TYPE_UDP        1
```

传输层协议选择决定了测试的数据传输方式。TCP 提供可靠连接，适合测试实际吞吐量；UDP 提供无连接服务，适合测试最大丢包率和抖动性能。

### 运行模式标志

```c
#define IPERF_FLAG_CLIENT           (1)
#define IPERF_FLAG_SERVER           (1 << 1)
#define IPERF_FLAG_TCP              (1 << 2)
#define IPERF_FLAG_UDP              (1 << 3)
#define IPERF_FLAG_DUAL             (1 << 4)
```

标志位用于配置 iPerf 的运行模式，可通过位运算组合多种模式：

| 标志 | 值 | 说明 |
|------|-----|------|
| IPERF_FLAG_CLIENT | 0x01 | 客户端模式，作为测试发起端 |
| IPERF_FLAG_SERVER | 0x02 | 服务端模式，接收测试数据 |
| IPERF_FLAG_TCP | 0x04 | 使用 TCP 协议传输 |
| IPERF_FLAG_UDP | 0x08 | 使用 UDP 协议传输 |
| IPERF_FLAG_DUAL | 0x10 | 双工模式，同时进行收发测试 |

### 默认配置参数

```c
#define IPERF_DEFAULT_PORT          5001
#define IPERF_DEFAULT_INTERVAL      1
#define IPERF_DEFAULT_TIME          10
#define IPERF_DEFAULT_NO_BW_LIMIT   -1
```

| 参数 | 值 | 说明 |
|------|-----|------|
| IPERF_DEFAULT_PORT | 5001 | 默认监听/连接端口 |
| IPERF_DEFAULT_INTERVAL | 1 | 报告间隔，单位秒 |
| IPERF_DEFAULT_TIME | 10 | 测试持续时间，单位秒 |
| IPERF_DEFAULT_NO_BW_LIMIT | -1 | 无带宽限制 |

### 任务配置参数

```c
#define IPERF_TRAFFIC_TASK_NAME     "iperf_traffic"
#define IPERF_TRAFFIC_TASK_PRIORITY 10
#define IPERF_TRAFFIC_TASK_STACK    2048
```

iPerf 使用独立任务处理流量收发，任务名称为 `iperf_traffic`，优先级为 10（相对值），栈大小为 2048 字节。

### 缓冲区长度配置

```c
#define IPERF_UDP_TX_LEN            (1470)
#define IPERF_UDP_RX_LEN            (1470)
#define IPERF_TCP_TX_LEN            (4 << 10)
#define IPERF_TCP_RX_LEN            (4 << 10)
```

| 参数 | 值 | 说明 |
|------|-----|------|
| IPERF_UDP_TX_LEN | 1470 | UDP 发送缓冲区，1470 字节符合 MTU |
| IPERF_UDP_RX_LEN | 1470 | UDP 接收缓冲区 |
| IPERF_TCP_TX_LEN | 4096 | TCP 发送缓冲区，4KB |
| IPERF_TCP_RX_LEN | 4096 | TCP 接收缓冲区，4KB |

UDP 缓冲区设置为 1470 字节，这是考虑到以太网 MTU（1500 字节）减去 IP 头（20 字节）和 UDP 头（8 字节）后的最佳 payload 大小。

### 其他配置参数

```c
#define IPERF_MAX_DELAY             64
#define IPERF_SOCKET_RX_TIMEOUT     10
#define IPERF_SOCKET_ACCEPT_TIMEOUT 5
```

| 参数 | 说明 |
|------|------|
| IPERF_MAX_DELAY | UDP 模式最大延迟容忍值 |
| IPERF_SOCKET_RX_TIMEOUT | Socket 接收超时时间 |
| IPERF_SOCKET_ACCEPT_TIMEOUT | 服务端接受连接超时时间 |

## 标志位操作宏

```c
#define IPERF_FLAG_SET(cfg, flag)   ((cfg) |= (flag))
#define IPERF_FLAG_CLR(cfg, flag)   ((cfg) &= (~(flag)))
```

这两个宏用于操作配置结构中的标志位字段：

- `IPERF_FLAG_SET(cfg, flag)`：将 cfg 中的 flag 标志位置 1
- `IPERF_FLAG_CLR(cfg, flag)`：将 cfg 中的 flag 标志位清零

使用示例：

```c
iperf_cfg_t cfg;
cfg.flag = 0;

// 设置为客户端模式，使用 TCP
IPERF_FLAG_SET(cfg.flag, IPERF_FLAG_CLIENT);
IPERF_FLAG_SET(cfg.flag, IPERF_FLAG_TCP);

// 清除 UDP 标志（如有）
IPERF_FLAG_CLR(cfg.flag, IPERF_FLAG_UDP);
```

## 配置结构体

```c
typedef struct {
    uint32_t flag;
    union {
        uint32_t destination_ip4;
        char *destination_ip6;
    };
    union {
        uint32_t source_ip4;
        char *source_ip6;
    };
    uint8_t type;
    uint16_t dport;
    uint16_t sport;
    uint32_t interval;
    uint32_t time;
    uint16_t len_buf;
    int32_t bw_lim;
    uint8_t tos;
    uint8_t traffic_task_priority;
    uint32_t num_bytes;
} iperf_cfg_t;
```

| 字段 | 类型 | 说明 |
|------|------|------|
| flag | uint32_t | 运行模式标志位组合 |
| destination_ip4 | uint32_t | 目标 IPv4 地址（网络字节序） |
| destination_ip6 | char* | 目标 IPv6 地址字符串 |
| source_ip4 | uint32_t | 源 IPv4 地址（网络字节序） |
| source_ip6 | char* | 源 IPv6 地址字符串 |
| type | uint8_t | IP 地址类型，IPv4 或 IPv6 |
| dport | uint16_t | 目标端口号 |
| sport | uint16_t | 源端口号 |
| interval | uint32_t | 报告间隔时间（秒） |
| time | uint32_t | 测试持续时间（秒） |
| len_buf | uint16_t | 缓冲区长度 |
| bw_lim | int32_t | 带宽限制（-1 表示无限制） |
| tos | uint8_t | 服务类型字段 |
| traffic_task_priority | uint8_t | 流量任务优先级 |
| num_bytes | uint32_t | 传输的总字节数 |

## API 接口

### 启动 iperf 测试

```c
int iperf_start(iperf_cfg_t *cfg);
```

**参数说明：**

- `cfg`：指向 iperf_cfg_t 配置结构体的指针

**返回值：**

- 0：成功启动
- 负值：启动失败

**功能描述：**

根据配置参数启动 iPerf 测试。测试可以运行在客户端或服务端模式，TCP 或 UDP 协议。该函数会创建独立的 traffic 任务来处理数据传输。

### 停止 iperf 测试

```c
int iperf_stop(void);
```

**返回值：**

- 0：成功停止
- 负值：停止失败

**功能描述：**

停止正在运行的 iPerf 测试，释放相关资源。

## 客户端模式

客户端模式用于主动向 iPerf 服务端发起测试请求，测量从本设备到服务端的网络吞吐量。

### 客户端配置要点

1. **设置客户端标志**：配置 `IPERF_FLAG_CLIENT` 标志
2. **指定目标地址**：设置目标服务端的 IP 地址和端口
3. **选择传输协议**：TCP 或 UDP，根据测试需求选择
4. **设置测试参数**：测试时长、报告间隔、带宽限制等

### TCP 客户端示例

```c
#include "iperf.h"
#include <stdint.h>

void iperf_tcp_client_example(void)
{
    iperf_cfg_t cfg;
    
    // 清零配置结构
    memset(&cfg, 0, sizeof(cfg));
    
    // 设置为客户端模式，使用 TCP
    cfg.flag = IPERF_FLAG_CLIENT | IPERF_FLAG_TCP;
    
    // 设置目标 IPv4 地址（假设为 192.168.1.100）
    cfg.type = IPERF_IP_TYPE_IPV4;
    cfg.destination_ip4 = 0x6401A8C0;  // 192.168.1.100 网络字节序
    
    // 设置端口
    cfg.dport = IPERF_DEFAULT_PORT;  // 5001
    
    // 设置测试参数
    cfg.interval = IPERF_DEFAULT_INTERVAL;  // 1 秒
    cfg.time = IPERF_DEFAULT_TIME;          // 10 秒
    
    // 设置缓冲区
    cfg.len_buf = IPERF_TCP_TX_LEN;  // 4096 字节
    
    // 设置带宽限制（-1 表示无限制）
    cfg.bw_lim = IPERF_DEFAULT_NO_BW_LIMIT;
    
    // 设置任务优先级
    cfg.traffic_task_priority = IPERF_TRAFFIC_TASK_PRIORITY;
    
    // 启动 TCP 客户端测试
    int ret = iperf_start(&cfg);
    if (ret == 0) {
        printf("iPerf TCP client started\r\n");
    } else {
        printf("iPerf TCP client failed: %d\r\n", ret);
    }
}
```

### UDP 客户端示例

```c
void iperf_udp_client_example(void)
{
    iperf_cfg_t cfg;
    
    memset(&cfg, 0, sizeof(cfg));
    
    // 设置为客户端模式，使用 UDP
    cfg.flag = IPERF_FLAG_CLIENT | IPERF_FLAG_UDP;
    
    // 设置目标地址
    cfg.type = IPERF_IP_TYPE_IPV4;
    cfg.destination_ip4 = 0x6401A8C0;  // 192.168.1.100
    
    // 设置端口
    cfg.dport = IPERF_DEFAULT_PORT;
    
    // 设置测试参数
    cfg.interval = IPERF_DEFAULT_INTERVAL;
    cfg.time = IPERF_DEFAULT_TIME;
    
    // UDP 缓冲区设置
    cfg.len_buf = IPERF_UDP_TX_LEN;  // 1470 字节
    
    // 限制带宽为 10 Mbps
    cfg.bw_lim = 10000;
    
    // 设置 TOS
    cfg.tos = 0;
    
    // 启动 UDP 客户端测试
    int ret = iperf_start(&cfg);
    if (ret == 0) {
        printf("iPerf UDP client started\r\n");
    } else {
        printf("iPerf UDP client failed: %d\r\n", ret);
    }
}
```

## 服务端模式

服务端模式用于接收来自 iPerf 客户端的测试数据，被动等待连接并报告接收性能。

### 服务端配置要点

1. **设置服务端标志**：配置 `IPERF_FLAG_SERVER` 标志
2. **绑定端口**：设置本地监听端口
3. **选择协议**：TCP 需要 accept 连接，UDP 直接接收数据
4. **配置报告**：设置报告间隔和时长

### TCP 服务端示例

```c
void iperf_tcp_server_example(void)
{
    iperf_cfg_t cfg;
    
    memset(&cfg, 0, sizeof(cfg));
    
    // 设置为服务端模式，使用 TCP
    cfg.flag = IPERF_FLAG_SERVER | IPERF_FLAG_TCP;
    
    // 设置本地监听端口
    cfg.dport = IPERF_DEFAULT_PORT;
    
    // 设置报告间隔
    cfg.interval = IPERF_DEFAULT_INTERVAL;
    
    // 设置缓冲区大小
    cfg.len_buf = IPERF_TCP_RX_LEN;  // 4096 字节
    
    // 启动 TCP 服务端
    int ret = iperf_start(&cfg);
    if (ret == 0) {
        printf("iPerf TCP server started on port %d\r\n", IPERF_DEFAULT_PORT);
    } else {
        printf("iPerf TCP server failed: %d\r\n", ret);
    }
}
```

### UDP 服务端示例

```c
void iperf_udp_server_example(void)
{
    iperf_cfg_t cfg;
    
    memset(&cfg, 0, sizeof(cfg));
    
    // 设置为服务端模式，使用 UDP
    cfg.flag = IPERF_FLAG_SERVER | IPERF_FLAG_UDP;
    
    // 设置监听端口
    cfg.dport = IPERF_DEFAULT_PORT;
    
    // 设置报告间隔
    cfg.interval = IPERF_DEFAULT_INTERVAL;
    
    // UDP 接收缓冲区
    cfg.len_buf = IPERF_UDP_RX_LEN;  // 1470 字节
    
    // 启动 UDP 服务端
    int ret = iperf_start(&cfg);
    if (ret == 0) {
        printf("iPerf UDP server started on port %d\r\n", IPERF_DEFAULT_PORT);
    } else {
        printf("iPerf UDP server failed: %d\r\n", ret);
    }
}
```

## 双工模式

双工模式（Dual Mode）允许同时进行双向测试，即在测试下行带宽的同时测试上行带宽。

```c
void iperf_dual_mode_example(void)
{
    iperf_cfg_t cfg;
    
    memset(&cfg, 0, sizeof(cfg));
    
    // 设置双工模式标志
    cfg.flag = IPERF_FLAG_CLIENT | IPERF_FLAG_TCP | IPERF_FLAG_DUAL;
    
    // 设置目标地址
    cfg.type = IPERF_IP_TYPE_IPV4;
    cfg.destination_ip4 = 0x6401A8C0;  // 192.168.1.100
    
    // 设置端口
    cfg.dport = IPERF_DEFAULT_PORT;
    
    // 测试时间
    cfg.interval = IPERF_DEFAULT_INTERVAL;
    cfg.time = IPERF_DEFAULT_TIME;
    
    // 缓冲区
    cfg.len_buf = IPERF_TCP_TX_LEN;
    
    // 无带宽限制
    cfg.bw_lim = IPERF_DEFAULT_NO_BW_LIMIT;
    
    // 启动双工测试
    int ret = iperf_start(&cfg);
    if (ret == 0) {
        printf("iPerf dual mode started\r\n");
    }
}
```

## Wi-Fi 吞吐量关系

iperf 测试结果与 Wi-Fi 吞吐量性能密切相关，以下是主要关系说明：

### 理论带宽与实际吞吐量

Wi-Fi 理论带宽取决于使用的协议标准：

| Wi-Fi 标准 | 频段 | 最大理论带宽 | 典型实际吞吐量 |
|------------|------|-------------|----------------|
| 802.11n | 2.4GHz | 600 Mbps | 200-400 Mbps |
| 802.11ac | 5GHz | 6.9 Gbps | 500-1000 Mbps |
| 802.11ax (Wi-Fi 6) | 2.4/5GHz | 9.6 Gbps | 600-1200 Mbps |

实际吞吐量通常为理论值的 30%-70%，受到多种因素影响。

### 影响吞吐量的因素

1. **信号强度**：RSSI 值越低，物理层速率越低，吞吐量越小
2. **干扰程度**：同信道干扰导致重传增加，有效吞吐量下降
3. **MCS 速率**：不同的调制编码方式影响物理层速率
4. **帧开销**：802.11 帧结构包含大量控制开销
5. **窗口大小**：TCP/UDP 窗口大小影响传输效率
6. **分片与聚合**：MSDU 聚合和 MPDU 聚合影响效率

### 测试建议

- **测试环境**：选择干扰较少的信道进行基准测试
- **测试时长**：每次测试至少 10 秒，以获得稳定平均值
- **协议选择**：TCP 测试适合评估实际可用带宽，UDP 测试适合评估极限能力
- **方向选择**：分别测试上行（设备发送）和下行（设备接收）性能
- **包大小**：默认 1470 字节（UDP）或 4096 字节（TCP）可获得最佳性能

### 性能评估指标

| 指标 | 说明 | 评估要点 |
|------|------|----------|
| 吞吐量 | 单位时间内传输的数据量 | 值越高越好 |
| 抖动 | UDP 包延迟的变化范围 | 值越低越稳定 |
| 丢包率 | UDP 传输中丢失包的比例 | 值越低越可靠 |
| 延迟 | 数据从发送到接收的时间 | 值越低响应越快 |

## 典型应用场景

### 场景一：Wi-Fi 连接性能验证

在设备成功连接 Wi-Fi 网络后，使用 iperf 进行吞吐量验证：

```c
void wifi_performance_test(void)
{
    // 假设 Wi-Fi 已连接
    printf("Starting Wi-Fi performance test...\r\n");
    
    // 先启动服务端接收测试
    iperf_cfg_t server_cfg;
    memset(&server_cfg, 0, sizeof(server_cfg));
    server_cfg.flag = IPERF_FLAG_SERVER | IPERF_FLAG_TCP;
    server_cfg.dport = IPERF_DEFAULT_PORT;
    server_cfg.interval = IPERF_DEFAULT_INTERVAL;
    server_cfg.len_buf = IPERF_TCP_RX_LEN;
    iperf_start(&server_cfg);
    
    // 等待服务端启动
    vTaskDelay(pdMS_TO_TICKS(500));
    
    // 启动客户端发起测试
    iperf_cfg_t client_cfg;
    memset(&client_cfg, 0, sizeof(client_cfg));
    client_cfg.flag = IPERF_FLAG_CLIENT | IPERF_FLAG_TCP;
    client_cfg.type = IPERF_IP_TYPE_IPV4;
    client_cfg.destination_ip4 = 0x0100007F;  // 127.0.0.1 或服务器 IP
    client_cfg.dport = IPERF_DEFAULT_PORT;
    client_cfg.interval = IPERF_DEFAULT_INTERVAL;
    client_cfg.time = IPERF_DEFAULT_TIME;
    client_cfg.len_buf = IPERF_TCP_TX_LEN;
    client_cfg.bw_lim = IPERF_DEFAULT_NO_BW_LIMIT;
    iperf_start(&client_cfg);
    
    // 等待测试完成
    vTaskDelay(pdMS_TO_TICKS(client_cfg.time * 1000 + 2000));
    
    printf("Performance test completed\r\n");
}
```

### 场景二：不同距离下的性能对比

在同一网络环境下，通过改变设备与 AP 的距离，评估覆盖范围对吞吐量的影响：

```c
void range_performance_test(const char *server_ip)
{
    iperf_cfg_t cfg;
    memset(&cfg, 0, sizeof(cfg));
    
    cfg.flag = IPERF_FLAG_CLIENT | IPERF_FLAG_UDP;
    cfg.type = IPERF_IP_TYPE_IPV4;
    cfg.destination_ip4 = ipaddr_aton(server_ip);
    cfg.dport = IPERF_DEFAULT_PORT;
    cfg.interval = IPERF_DEFAULT_INTERVAL;
    cfg.time = 5;  // 短时间测试
    cfg.len_buf = IPERF_UDP_TX_LEN;
    
    // 以不同带宽限制模拟不同距离下的性能
    int bandwidths[] = {50000, 20000, 10000, 5000};  // 50/20/10/5 Mbps
    
    for (int i = 0; i < 4; i++) {
        cfg.bw_lim = bandwidths[i];
        printf("Testing with bandwidth limit: %d Kbps\r\n", bandwidths[i]);
        iperf_start(&cfg);
        vTaskDelay(pdMS_TO_TICKS(6000));
    }
}
```

### 场景三：持续压力测试

通过长时间运行 iperf 测试，观察吞吐量的稳定性：

```c
void stress_test_example(const char *server_ip)
{
    iperf_cfg_t cfg;
    memset(&cfg, 0, sizeof(cfg));
    
    cfg.flag = IPERF_FLAG_CLIENT | IPERF_FLAG_TCP;
    cfg.type = IPERF_IP_TYPE_IPV4;
    cfg.destination_ip4 = ipaddr_aton(server_ip);
    cfg.dport = IPERF_DEFAULT_PORT;
    cfg.interval = IPERF_DEFAULT_INTERVAL;
    cfg.time = 60;  // 1 分钟测试
    cfg.len_buf = IPERF_TCP_TX_LEN;
    cfg.bw_lim = IPERF_DEFAULT_NO_BW_LIMIT;
    
    printf("Starting 60-second stress test...\r\n");
    iperf_start(&cfg);
}
```

## 注意事项

### 防火墙配置

在运行 iPerf 测试前，确保测试设备的防火墙允许 5001 端口的入站和出站流量。对于 Linux 系统，可使用以下命令：

```bash
# 开放 5001 端口
sudo ufw allow 5001
```

### 网络兼容性

- 确保测试两端（设备与 PC）连接至同一网络或可互通的网络
- IPv6 测试需要网络环境支持 IPv6 协议栈
- 跨 NAT 测试需要端口映射配置

### 资源占用

iPerf 流量任务会占用一定的 CPU 和内存资源：

- 任务栈：2048 字节
- 缓冲区：TCP 模式 4KB × 2，UDP 模式 1470 字节 × 2
- CPU 开销：主要消耗在数据拷贝和协议处理上

### 错误处理

```c
int ret = iperf_start(&cfg);
if (ret < 0) {
    switch (ret) {
        case -1:
            printf("Invalid configuration\r\n");
            break;
        case -2:
            printf("Socket creation failed\r\n");
            break;
        case -3:
            printf("Task creation failed\r\n");
            break;
        default:
            printf("Unknown error: %d\r\n", ret);
            break;
    }
}
```

## 参考

- Bouffalo SDK iperf 组件源码：`components/iperf/iperf.h`
- lwIP TCP/IP 协议栈文档
- IETF RFC 9000（QUIC 协议定义）
- Wi-Fi Alliance 802.11 标准文档
