# BLE Mesh API 参考

> 来源文件：`components/network/ble/blemesh/src/mesh.h` 等  
> BLE Mesh 是基于 BLE 5.0 的低功耗 mesh 网络协议，适用于大规模物联网设备组网。

## 概述

BLE Mesh 网络架构：

```
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Node A  │────▶│ Node B  │────▶│ Node C  │
│(Relay)  │     │(Relay)  │     │(Leaf)   │
└─────────┘     └─────────┘     └─────────┘
```

**角色类型**：Provisioner（配网器）、Node（节点）、Relay（转发）、Proxy（代理）、Friend（友节点）、Low Power（低功耗节点）

---

## 初始化

### `ble_mesh_init`

初始化 BLE Mesh 协议栈。

```c
int ble_mesh_init(void);
```

---

## 配置

### `ble_mesh_prov_enable`

使能 BLE Mesh Provisioning。

```c
int ble_mesh_prov_enable(void);
```

---

### `ble_mesh_config_enable`

使能 BLE Mesh 配置模型。

```c
int ble_mesh_config_enable(void);
```

---

### `ble_mesh_app_key_add`

添加应用密钥。

```c
int ble_mesh_app_key_add(uint16_t net_idx, uint16_t app_idx, const uint8_t key[16]);
```

| 参数 | 说明 |
|------|------|
| `net_idx` | 网络密钥索引 |
| `app_idx` | 应用密钥索引 |
| `key` | 128 位密钥 |

---

## 发送与接收

### `ble_mesh_model_publish

发布消息（模型主动发送）。

```c
int ble_mesh_model_publish(uint16_t elem_idx, uint16_t mod_id,
                            uint16_t opcode, size_t len, uint8_t *data);
```

---

### `ble_mesh_model_send`

发送模型消息。

```c
int ble_mesh_model_send(uint16_t dst, uint16_t app_idx,
                        uint16_t elem_idx, uint16_t mod_id,
                        uint16_t opcode, size_t len, const uint8_t *data);
```

| 参数 | 说明 |
|------|------|
| `dst` | 目标单播/组播地址 |
| `app_idx` | 应用密钥索引 |
| `elem_idx` | 元素索引 |
| `mod_id` | 模型 ID |
| `opcode` | 操作码 |
| `data` | 消息载荷 |

---

## 订阅

### `ble_mesh_model_sub_add`

添加订阅地址。

```c
int ble_mesh_model_sub_add(uint16_t elem_idx, uint16_t sub_addr,
                            uint16_t mod_id, uint16_t cid);
```

---

## 通用属性

### `ble_mesh_gen_onoff_get`

获取通用开关状态（OnOff Server）。

```c
int ble_mesh_gen_onoff_get(uint16_t addr, uint8_t *onoff);
```

---

### `ble_mesh_gen_onoff_set`

设置通用开关状态。

```c
int ble_mesh_gen_onoff_set(uint16_t addr, uint8_t onoff, uint8_t ack);
```

| 参数 | 说明 |
|------|------|
| `addr` | 目标节点地址 |
| `onoff` | `0` = 关，`1` = 开 |
| `ack` | `1` = 请求应答 |

---

### `ble_mesh_gen_level_get` / `ble_mesh_gen_level_set`

获取/设置通用级别（0~65535）。

```c
int ble_mesh_gen_level_get(uint16_t addr, int16_t *level);
int ble_mesh_gen_level_set(uint16_t addr, int16_t level, uint8_t ack);
```

---

## 配置客户端操作

### `ble_mesh_cfg_client_get`

获取节点配置。

```c
int ble_mesh_cfg_client_get(uint16_t dst, uint16_t elem_idx, uint8_t opcode);
```

---

### `ble_mesh_cfg_client_set`

设置节点配置。

```c
int ble_mesh_cfg_client_set(uint16_t dst, uint16_t elem_idx,
                             uint8_t opcode, size_t len, const uint8_t *data);
```

---

## 网关/Proxy

### `ble_mesh_proxy_filter_set`

设置 Proxy 过滤器类型。

```c
int ble_mesh_proxy_filter_set(uint16_t net_idx, uint8_t filter_type);
```

> `filter_type`：`PROXY_FILTER_WHITELIST` / `PROXY_FILTER_BLACKLIST`

---

## 健康相关

### `ble_mesh_health_fault_get`

获取健康故障状态。

```c
int ble_mesh_health_fault_get(uint16_t addr, uint8_t test_id, uint8_t *faults);
```

---

### `ble_mesh_health_attention_set`

设置节点关注时间。

```c
int ble_mesh_health_attention_set(uint16_t addr, uint8_t attention);
```

---

## 使用示例

```c
#include "mesh.h"

// 初始化 BLE Mesh
ble_mesh_init();

// 作为 Provisioner（配网器）
ble_mesh_prov_enable();
ble_mesh_config_enable();

// 添加密钥
uint8_t net_key[16] = {0x00, 0x11, 0x22, /*...*/};
uint8_t app_key[16] = {0xAA, 0xBB, 0xCC, /*...*/};
ble_mesh_app_key_add(0, 0, app_key);

// 控制灯节点（单播地址 0x0002）的开关
ble_mesh_gen_onoff_set(0x0002, 1, 1); // 开灯并请求应答

// 获取灯状态
uint8_t state;
ble_mesh_gen_onoff_get(0x0002, &state);
printf("Light state: %s\r\n", state ? "ON" : "OFF");
```

## 常用模型 ID

| 模型 | ID | 说明 |
|------|-----|------|
| Generic OnOff Server | `0x1000` | 开关控制 |
| Generic Level Server | `0x1002` | 级别控制 |
| Light Lightness Server | `0x1300` | 亮度控制 |
| Light CTL Server | `0x1303` | 色温控制 |
| Sensor Server | `0x1100` | 传感器数据 |
| Time Server | `0x1200` | 时间服务 |
| Configuration Server | `0x0000` | 配置模型（必须） |
| Health Server | `0x0002` | 健康模型（必须） |
