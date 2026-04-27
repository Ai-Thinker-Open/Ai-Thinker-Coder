# BLUFI 配网 API 参考

> 来源文件：`components/network/blufi/blufi_api.h`  
> BLUFI 是安信可基于 BLE 通道实现 Wi-Fi 配网和控制的协议，手机 APP 通过 BLE 发送 SSID/密码，BL602 接收后自动连接指定热点。

---

## 概述

BLUFI 配网流程：

```
手机 APP (BLE) ──▶ BL602 (BLUFI) ──▶ 连接指定 Wi-Fi 热点
                         │
                         ├── 解析 SSID/密码
                         ├── 配置 AP 模式（可选）
                         ├── 连接 Wi-Fi
                         └── 上报连接结果
```

**事件类型**：

| 事件 | 说明 |
|------|------|
| `AXK_BLUFI_EVENT_INIT_FINISH` | BLUFI 初始化完成 |
| `AXK_BLUFI_EVENT_BLE_CONNECT` | 手机 BLE 连接成功 |
| `AXK_BLUFI_EVENT_BLE_DISCONNECT` | 手机 BLE 断开 |
| `AXK_BLUFI_EVENT_RECV_STA_SSID` | 收到 STA 的 SSID |
| `AXK_BLUFI_EVENT_RECV_STA_PASSWD` | 收到 STA 的密码 |
| `AXK_BLUFI_EVENT_REQ_CONNECT_TO_AP` | 请求连接热点 |
| `AXK_BLUFI_EVENT_GET_WIFI_STATUS` | 查询 Wi-Fi 状态 |
| `AXK_BLUFI_EVENT_RECV_SOFTAP_SSID` | 收到 AP 的 SSID |
| `AXK_BLUFI_EVENT_RECV_CUSTOM_DATA` | 收到自定义数据 |

---

## 类型定义

### `_blufi_cb_event_t` — BLUFI 事件类型

```c
typedef enum {
    AXK_BLUFI_EVENT_INIT_FINISH = 0,
    AXK_BLUFI_EVENT_DEINIT_FINISH,
    AXK_BLUFI_EVENT_BLE_CONNECT,
    AXK_BLUFI_EVENT_BLE_DISCONNECT,
    AXK_BLUFI_EVENT_SET_WIFI_OPMODE,
    AXK_BLUFI_EVENT_REQ_CONNECT_TO_AP,
    AXK_BLUFI_EVENT_REQ_DISCONNECT_FROM_AP,
    AXK_BLUFI_EVENT_GET_WIFI_STATUS,
    AXK_BLUFI_EVENT_RECV_STA_BSSID,
    AXK_BLUFI_EVENT_RECV_STA_SSID,
    AXK_BLUFI_EVENT_RECV_STA_PASSWD,
    AXK_BLUFI_EVENT_RECV_SOFTAP_SSID,
    AXK_BLUFI_EVENT_RECV_SOFTAP_PASSWD,
    AXK_BLUFI_EVENT_RECV_CUSTOM_DATA,
    // ... 更多事件
} _blufi_cb_event_t;
```

### `_blufi_callbacks_t` — BLUFI 回调结构

```c
typedef struct {
    _blufi_event_cb_t             event_cb;             // 事件回调
    _blufi_negotiate_data_handler_t negotiate_data_handler; // 密钥协商
    _blufi_encrypt_func_t         encrypt_func;         // 加密函数
    _blufi_decrypt_func_t         decrypt_func;         // 解密函数
    _blufi_checksum_func_t        checksum_func;        // 校验和函数
} _blufi_callbacks_t;
```

---

## 函数接口

### `axk_blufi_register_callbacks`

注册 BLUFI 回调函数。

```c
int axk_blufi_register_callbacks(_blufi_callbacks_t *callbacks);
```

| 参数 | 说明 |
|------|------|
| `callbacks` | 回调函数结构体指针 |

---

### `axk_blufi_profile_init`

初始化 BLUFI 协议层。

```c
int axk_blufi_profile_init(void);
```

---

### `axk_blufi_profile_deinit`

去初始化 BLUFI。

```c
int axk_blufi_profile_deinit(void);
```

---

### `axk_blufi_send_wifi_conn_report`

向手机上报 Wi-Fi 连接状态。

```c
int axk_blufi_send_wifi_conn_report(wifi_mode_t opmode,
                                     axk_blufi_sta_conn_state_t sta_conn_state,
                                     uint8_t softap_conn_num,
                                     axk_blufi_extra_info_t *extra_info);
```

| 参数 | 说明 |
|------|------|
| `opmode` | Wi-Fi 模式（STA/AP） |
| `sta_conn_state` | 连接状态（`_BLUFI_STA_CONN_SUCCESS` / `_BLUFI_STA_CONN_FAIL`） |
| `softap_conn_num` | AP 已连接终端数 |
| `extra_info` | 额外信息（SSID 等） |

---

### `axk_blufi_send_wifi_list`

向手机发送 Wi-Fi 列表。

```c
int axk_blufi_send_wifi_list(uint16_t apCount, _blufi_ap_record_t *list);
```

---

### `axk_blufi_send_error_info`

上报 BLUFI 错误信息。

```c
int axk_blufi_send_error_info(_blufi_error_state_t state);
```

---

### `axk_blufi_send_custom_data`

向手机发送自定义数据。

```c
int axk_blufi_send_custom_data(uint8_t *data, uint32_t data_len);
```

---

## 使用示例

```c
#include "blufi_api.h"

static char s_sta_ssid[32];
static char s_sta_passwd[64];

// BLUFI 事件回调
static void blufi_event_cb(_blufi_cb_event_t event, _blufi_cb_param_t *param)
{
    switch (event) {
    case AXK_BLUFI_EVENT_INIT_FINISH:
        printf("BLUFI init done\r\n");
        break;

    case AXK_BLUFI_EVENT_BLE_CONNECT:
        printf("Phone BLE connected\r\n");
        break;

    case AXK_BLUFI_EVENT_BLE_DISCONNECT:
        printf("Phone BLE disconnected\r\n");
        break;

    case AXK_BLUFI_EVENT_RECV_STA_SSID:
        memcpy(s_sta_ssid, param->sta_ssid.ssid, param->sta_ssid.ssid_len);
        s_sta_ssid[param->sta_ssid.ssid_len] = '\0';
        printf("STA SSID: %s\r\n", s_sta_ssid);
        break;

    case AXK_BLUFI_EVENT_RECV_STA_PASSWD:
        memcpy(s_sta_passwd, param->sta_passwd.passwd, param->sta_passwd.passwd_len);
        s_sta_passwd[param->sta_passwd.passwd_len] = '\0';
        printf("STA PASSWD received\r\n");
        break;

    case AXK_BLUFI_EVENT_REQ_CONNECT_TO_AP:
        printf("Connecting to AP: %s\r\n", s_sta_ssid);
        // 使用 BLUFI 收到的 SSID/密码连接 Wi-Fi
        wifi_sta_connect(s_sta_ssid, s_sta_passwd);
        // 上报连接结果
        axk_blufi_send_wifi_conn_report(WIFI_MODE_STA,
                                         _BLUFI_STA_CONN_SUCCESS,
                                         0, NULL);
        break;

    case AXK_BLUFI_EVENT_GET_WIFI_STATUS:
        // 上报当前 Wi-Fi 状态
        axk_blufi_send_wifi_conn_report(WIFI_MODE_STA,
                                         _BLUFI_STA_CONN_SUCCESS,
                                         0, NULL);
        break;

    case AXK_BLUFI_EVENT_RECV_CUSTOM_DATA:
        printf("Custom data: %.*s\r\n",
               param->custom_data.data_len,
               param->custom_data.data);
        break;
    }
}

// 加密/解密/校验和回调（可使用默认实现）
static void blufi_negotiate_data_handler(uint8_t *data, int len,
                                          uint8_t **output, int *out_len,
                                          bool *need_free)
{
    // 默认实现
    *need_free = false;
}

static int blufi_encrypt(uint8_t iv8, uint8_t *crypt_data, int crypt_len)
{
    return crypt_len; // 默认不加密
}

static int blufi_decrypt(uint8_t iv8, uint8_t *crypt_data, int crypt_len)
{
    return crypt_len; // 默认不解密
}

static uint16_t blufi_checksum(uint8_t iv8, uint8_t *data, int len)
{
    return 0; // 默认校验和
}

void blufi_app_init(void)
{
    _blufi_callbacks_t callbacks = {
        .event_cb = blufi_event_cb,
        .negotiate_data_handler = blufi_negotiate_data_handler,
        .encrypt_func = blufi_encrypt,
        .decrypt_func = blufi_decrypt,
        .checksum_func = blufi_checksum,
    };

    axk_blufi_register_callbacks(&callbacks);
    axk_blufi_profile_init();
    printf("BLUFI initialized\r\n");
}
```
