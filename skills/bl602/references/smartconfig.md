# SmartConfig / AirKiss 配网 API 参考

> 来源文件：`components/network/smartconfig_airkiss/smartconfig.h`  
> SmartConfig 是乐鑫/安信可的 Wi-Fi 一键配网协议，AirKiss 是微信的配网协议。BL602 支持这两种方式通过广播包传输 SSID/密码。

---

## 概述

配网流程：

```
手机 APP (UDP广播) ──▶ BL602 监听信道1-13 ──▶ 解析 SSID/密码 ──▶ 连接热点
                              ↑
                         sniffer模式
```

**配网原理**：手机 APP 在 UDP 9999 端口发送包含 SSID/密码编码的广播包，BL602 通过 sniffer 模式抓取空口报文并解码。

---

## 类型定义

### `libwifi_frame` — 原始帧结构

```c
struct libwifi_frame {
    struct libwifi_frame_ctrl frame_control;  // 帧控制
    uint16_t duration;
    uint8_t  addr1[6];   // BSSID / Destination
    uint8_t  addr2[6];   // Source
    uint8_t  addr3[6];   // BSSID / Filtering
    struct libwifi_seq_control seq_control;
} __attribute__((packed));
```

### `libwifi_frame_ctrl` — 帧控制字段

```c
struct libwifi_frame_ctrl {
    unsigned int version : 2;   // 协议版本
    unsigned int type : 2;      // 帧类型
    unsigned int subtype : 4;   // 帧子类型
    struct libwifi_frame_ctrl_flags flags;
};
```

---

## 函数接口

### `wifi_smartconfig_v1_start`

启动 SmartConfig v1 配网。

```c
int wifi_smartconfig_v1_start(void);
```

**返回值**：`0` 成功，其他失败

> 调用后 BL602 进入 sniffer 模式，监听所有信道，等待手机 APP 发送配网数据。

---

### `wifi_smartconfig_v1_stop`

停止 SmartConfig 配网。

```c
int wifi_smartconfig_v1_stop(void);
```

**返回值**：`0` 成功，其他失败

> 配网成功后必须调用此函数恢复正常 Wi-Fi 模式。

---

## 使用示例

```c
#include "smartconfig.h"

// 配网任务
static void smartconfig_task(void *arg)
{
    printf("SmartConfig started, waiting for SSID...\r\n");

    // 启动配网
    int ret = wifi_smartconfig_v1_start();
    if (ret != 0) {
        printf("SmartConfig start failed: %d\r\n", ret);
        vTaskDelete(NULL);
        return;
    }

    // 等待配网成功（通常由 Wi-Fi 连接事件触发）
    // 实际项目中应在 Wi-Fi 连接回调中判断配网结果

    // 停止配网
    wifi_smartconfig_v1_stop();
    printf("SmartConfig stopped\r\n");

    vTaskDelete(NULL);
}

void app_main(void)
{
    // 系统初始化...

    // 创建配网任务（高优先级）
    xTaskCreate(smartconfig_task, "smartconfig", 4096, NULL, 5, NULL);
}
```

## 配网步骤（APP 端）

手机 APP 配网步骤（参考）：

1. 手机连接目标 Wi-Fi（用于发送广播）
2. APP 编码 SSID/密码到 UDP 数据包
3. APP 在每个信道上发送 UDP 广播包（9999 端口）
4. BL602 sniffer 模式接收并解码
5. BL602 连接目标热点
6. 上报连接结果（MQTT/TCP）

> **注意**：SmartConfig 依赖明文广播，安全性较低。生产环境建议使用 BLUFI（BLE 加密通道）配网。
