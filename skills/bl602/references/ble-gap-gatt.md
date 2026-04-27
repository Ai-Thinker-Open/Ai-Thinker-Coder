# BLE GAP/GATT API 参考

> 来源文件：`components/network/ble/blestack/src/host/hci_core.h` 等  
> BL602 使用开源 blestack 实现 BLE 5.0 Host 协议栈（GAP 和 GATT 层）。

---

## 概述

BLE 协议栈分层：

```
┌──────────────────────────────────────┐
│         Application (GATT Client/Server) │
├──────────────────────────────────────┤
│         GATT Server / Client          │
│  (Service/Characteristic/Descriptor)  │
├──────────────────────────────────────┤
│         GAP (Advertising/Connection)  │
│   (Broadcast/Connect/Discover/Security) │
├──────────────────────────────────────┤
│         HCI (Host-Controller Interface)│
├──────────────────────────────────────┤
│         BLE Controller (LL)           │
└──────────────────────────────────────┘
```

---

## GAP — 广播与连接

### 广播参数

```c
struct bt_le_adv_param {
    uint8_t id;          // 广播集 ID (BT_ID_DEFAULT = 0)
    uint8_t sid;         // 广播集 ID
    uint8_t freq;        // 广播信道 (BT_GAP_SCAN_FAST_INTERVAL = 0x0060)
    uint16_t interval_min;  // 最小广播间隔 (slots)
    uint16_t interval_max;  // 最大广播间隔 (slots)
    uint8_t  type;          // 广播类型
    uint8_t  chan_map;      // 信道 map
    uint8_t  filter_policy; // 广播过滤策略
    uint8_t  tier;          // 广播功率等级
};
```

**广播类型**（`type`）：

| 值 | 广播类型 | 说明 |
|----|---------|------|
| `BT_GAP_ADV_TYPE_ADV_IND` | 可连接广播 | 最常用 |
| `BT_GAP_ADV_TYPE_ADV_DIRECT_IND` | 定向广播 | 快速连接指定设备 |
| `BT_GAP_ADV_TYPE_ADV_SCAN_IND` | 可扫描广播 | 允许扫描请求 |
| `BT_GAP_ADV_TYPE_ADV_NONCONN_IND` | 不可连接广播 | 仅广播数据 |

### `bt_enable`

初始化 BLE 协议栈。

```c
int bt_enable(void);
```

> 调用其他 BLE API 前必须先调用此函数。

---

### `bt_le_adv_start`

开始广播。

```c
int bt_le_adv_start(const struct bt_le_adv_param *param,
                     const struct bt_data *ad,
                     size_t ad_len,
                     const struct bt_data *scan_rsp,
                     size_t scan_rsp_len);
```

| 参数 | 说明 |
|------|------|
| `param` | 广播参数 |
| `ad` | 广播数据（AD Structure） |
| `ad_len` | 广播数据数量 |
| `scan_rsp` | 扫描响应数据（可为空） |
| `scan_rsp_len` | 扫描响应数据数量 |

---

### `bt_le_adv_stop`

停止广播。

```c
int bt_le_adv_stop(void);
```

---

### `bt_le_scan_start`

开始扫描。

```c
int bt_le_scan_start(uint8_t scan_type,
                     const struct bt_le_scan_param *param);
```

---

### `bt_le_scan_stop`

停止扫描。

```c
int bt_le_scan_stop(void);
```

---

## GATT — 属性协议

### 通用属性

### `bt_gatt_service_register`

注册 GATT Service。

```c
uint8_t bt_gatt_service_register(const struct bt_gatt_service *svc);
```

---

### `bt_gatt_notify`

向客户端发送通知（无需响应）。

```c
int bt_gatt_notify(struct bt_conn *conn,
                   const struct bt_gatt_attr *attr,
                   const void *data,
                   uint16_t len);
```

---

### `bt_gatt_indicate`

向客户端发送指示（需要确认）。

```c
int bt_gatt_indicate(struct bt_conn *conn,
                     struct bt_gatt_attr *attr,
                     const void *data,
                     uint16_t len);
```

---

### `bt_gatt_read`

读取属性值。

```c
ssize_t bt_gatt_read(struct bt_conn *conn,
                      uint8_t flags,
                      const struct bt_gatt_attr *attr,
                      void *buf,
                      uint16_t buf_len,
                      uint16_t offset);
```

---

### `bt_gatt_write`

写入属性值（带响应）。

```c
int bt_gatt_write(struct bt_conn *conn,
                   uint8_t flags,
                   const struct bt_gatt_attr *attr,
                   const void *data,
                   uint16_t len,
                   bt_gatt_write_func_t func,
                   void *user_data);
```

---

### `bt_gatt_write_without_response`

写入属性值（无响应）。

```c
uint8_t bt_gatt_write_without_response(struct bt_conn *conn,
                                        const struct bt_gatt_attr *attr,
                                        const void *data,
                                        uint16_t len,
                                        bool sign);
```

---

## 连接管理

### `bt_conn_connect`

建立 BLE 连接。

```c
struct bt_conn *bt_conn_connect(const struct bt_conn_info *info);
```

---

### `bt_conn_disconnect`

断开连接。

```c
int bt_conn_disconnect(struct bt_conn *conn, uint8_t reason);
```

---

### `bt_conn_get_info`

获取连接信息。

```c
int bt_conn_get_info(const struct bt_conn *conn, struct bt_conn_info *info);
```

---

## 使用示例

### BLE 广播（从机）

```c
#include "bluetooth.h"

static const struct bt_data ad[] = {
    BT_DATA(BT_DATA_FLAGS, (uint8_t[]){ BT_LE_AD_LIMITED }, 1),
    BT_DATA(BT_DATA_NAME_COMPLETE, "BL602", 5),
};

static const struct bt_data sd[] = {
    BT_DATA(BT_DATA_UUID128_SOME, ...),
};

int ble_adv_start(void)
{
    struct bt_le_adv_param param = {
        .id = 0,
        .type = BT_GAP_ADV_TYPE_ADV_IND,
        .interval_min = 0x0020,
        .interval_max = 0x0020,
    };

    return bt_le_adv_start(&param, ad, ARRAY_SIZE(ad),
                           sd, ARRAY_SIZE(sd));
}
```

### GATT 服务注册

```c
static ssize_t read_temp(struct bt_conn *conn,
                         const struct bt_gatt_attr *attr,
                         void *buf, uint16_t len, uint16_t offset)
{
    uint16_t temp = read_temperature();
    return bt_gatt_attr_read(conn, attr, buf, len, offset, &temp, sizeof(temp));
}

static struct bt_gatt_attr attrs[] = {
    BT_GATT_PRIMARY_SERVICE(UUID_SENSOR_SVC),
    BT_GATT_CHARACTERISTIC(UUID_TEMP,
                           BT_GATT_CHRC_READ,
                           BT_GATT_PERM_READ,
                           read_temp, NULL, NULL),
};

static struct bt_gatt_service sensor_svc =
    BT_GATT_SERVICE(attrs);

bt_gatt_service_register(&sensor_svc);
```

### 发送通知

```c
extern struct bt_conn *conn;

static void notify_callback(struct bt_conn *conn,
                            struct bt_gatt_attr *attr,
                            void *context)
{
    // 值变化时通知客户端
    uint8_t data = get_sensor_value();
    bt_gatt_notify(conn, attr, &data, sizeof(data));
}
```
