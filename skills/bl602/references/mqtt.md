# MQTT API 参考

> 来源文件：`components/network/axk_protocol_stack/axk_mqtt/include/mqtt_client.h`  
> BL602 内置完整 MQTT 客户端，支持 MQTT over TCP/SSL/WebSocket，支持 QoS 0/1/2、LWT 遗嘱消息、自动重连。

---

## 概述

MQTT 协议栈基于 axk-mqtt，支持以下传输方式：

| 传输类型 | scheme | 说明 |
|---------|--------|------|
| MQTT over TCP | `mqtt://` | 普通 TCP 连接 |
| MQTT over SSL | `mqtts://` | TLS 加密连接 |
| MQTT over WS | `ws://` | WebSocket 明文 |
| MQTT over WSS | `wss://` | WebSocket TLS |

---

## 头文件

```c
#include "mqtt_client.h"
```

---

## 事件类型

### `axk_mqtt_event_id_t`

MQTT 事件枚举，在事件回调中区分处理：

```c
typedef enum {
    MQTT_EVENT_ANY = -1,
    MQTT_EVENT_ERROR = 0,          // 错误事件
    MQTT_EVENT_CONNECTED,          // 连接成功
    MQTT_EVENT_DISCONNECTED,       // 连接断开
    MQTT_EVENT_SUBSCRIBED,        // 订阅成功
    MQTT_EVENT_UNSUBSCRIBED,      // 取消订阅成功
    MQTT_EVENT_PUBLISHED,         // 发布成功
    MQTT_EVENT_DATA,              // 收到消息数据
    MQTT_EVENT_BEFORE_CONNECT,    // 连接前事件
    MQTT_EVENT_DELETED,           // 消息被删除（超时报错）
} axk_mqtt_event_id_t;
```

---

## 事件结构

### `axk_mqtt_event_t`

事件回调中携带的上下文信息：

```c
typedef struct {
    axk_mqtt_event_id_t event_id;       // 事件类型
    axk_mqtt_client_handle_t client;   // MQTT 客户端句柄
    void *user_context;                 // 用户上下文
    char *data;                         // 消息数据
    int data_len;                       // 数据长度
    int total_data_len;                 // 总长度（长消息分片）
    int current_data_offset;            // 当前数据偏移
    char *topic;                        // 消息主题
    int topic_len;                      // 主题长度
    int msg_id;                         // 消息 ID
    int session_present;                // 会话保持标志
    axk_mqtt_error_codes_t *error_handle; // 错误信息
    bool retain;                        // 保留消息标志
    int qos;                            // QoS 等级
    bool dup;                           // 重复标志
} axk_mqtt_event_t;
```

### 连接错误码

```c
typedef enum {
    MQTT_CONNECTION_ACCEPTED = 0,                   // 连接成功
    MQTT_CONNECTION_REFUSE_PROTOCOL,                // 协议错误
    MQTT_CONNECTION_REFUSE_ID_REJECTED,             // ID 被拒绝
    MQTT_CONNECTION_REFUSE_SERVER_UNAVAILABLE,      // 服务器不可用
    MQTT_CONNECTION_REFUSE_BAD_USERNAME,           // 用户名错误
    MQTT_CONNECTION_REFUSE_NOT_AUTHORIZED          // 密码错误
} axk_mqtt_connect_return_code_t;
```

---

## 客户端配置

### `axk_mqtt_client_config_t`

```c
typedef struct {
    mqtt_event_callback_t event_handle;      // 事件回调函数
    const char *host;                        // MQTT 服务器域名/IP
    const char *uri;                         // 完整 URI（覆盖 host/port）
    uint32_t port;                           // 服务器端口
    const char *client_id;                   // 客户端 ID（默认 BL602_XXXXXX）
    const char *username;                   // 用户名
    const char *password;                   // 密码
    const char *lwt_topic;                 // 遗嘱主题
    const char *lwt_msg;                   // 遗嘱消息
    int lwt_qos;                            // 遗嘱 QoS
    int lwt_retain;                         // 遗嘱保留
    int disable_clean_session;               // 禁用清会话（1=保持会话）
    int keepalive;                          // 保活间隔（秒，默认 120）
    bool disable_auto_reconnect;            // 禁用自动重连
    void *user_context;                     // 用户上下文
    int task_prio;                          // MQTT 任务优先级（默认 5）
    int task_stack;                         // 任务栈大小（默认 6144）
    int buffer_size;                        // 缓冲区大小（默认 1024）
    const char *cert_pem;                   // 服务器 CA 证书（PEM）
    size_t cert_len;                        // 证书长度
    const char *client_cert_pem;            // 客户端证书（双向认证）
    size_t client_cert_len;
    const char *client_key_pem;            // 客户端私钥
    size_t client_key_len;
    axk_mqtt_transport_t transport;         // 传输类型（TCP/SSL/WS/WSS）
    int reconnect_timeout_ms;               // 重连超时（默认 10s）
    const char **alpn_protos;              // ALPN 协议列表
    axk_mqtt_protocol_ver_t protocol_ver;  // MQTT 版本（3.1/3.1.1）
    bool skip_cert_common_name_check;       // 跳过 CN 验证
    bool disable_keepalive;                 // 禁用保活
    const char *path;                      // WebSocket 路径
} axk_mqtt_client_config_t;
```

---

## 函数接口

### `axk_mqtt_client_init`

创建并初始化 MQTT 客户端。

```c
axk_mqtt_client_handle_t axk_mqtt_client_init(const axk_mqtt_client_config_t *config);
```

| 参数 | 说明 |
|------|------|
| `config` | MQTT 配置结构体指针 |

**返回值**：成功返回客户端句柄，失败返回 NULL

---

### `axk_mqtt_client_start`

启动 MQTT 客户端（发起连接）。

```c
axk_err_t axk_mqtt_client_start(axk_mqtt_client_handle_t client);
```

---

### `axk_mqtt_client_reconnect`

强制重连。

```c
axk_err_t axk_mqtt_client_reconnect(axk_mqtt_client_handle_t client);
```

---

### `axk_mqtt_client_disconnect`

主动断开连接。

```c
axk_err_t axk_mqtt_client_disconnect(axk_mqtt_client_handle_t client);
```

---

### `axk_mqtt_client_stop`

停止 MQTT 任务。

```c
axk_err_t axk_mqtt_client_stop(axk_mqtt_client_handle_t client);
```

---

### `axk_mqtt_client_subscribe`

订阅主题。

```c
int axk_mqtt_client_subscribe(axk_mqtt_client_handle_t client,
                              const char *topic, int qos);
```

| 参数 | 说明 |
|------|------|
| `topic` | 订阅的主题（支持通配符 `+` `#`） |
| `qos` | QoS 等级（0/1/2） |

**返回值**：成功返回消息 ID，失败返回 -1

---

### `axk_mqtt_client_unsubscribe`

取消订阅。

```c
int axk_mqtt_client_unsubscribe(axk_mqtt_client_handle_t client,
                                const char *topic);
```

---

### `axk_mqtt_client_publish`

发布消息。

```c
int axk_mqtt_client_publish(axk_mqtt_client_handle_t client,
                             const char *topic,
                             const char *data, int len,
                             int qos, int retain);
```

| 参数 | 说明 |
|------|------|
| `topic` | 发布的主题 |
| `data` | 消息载荷 |
| `len` | 载荷长度（0=自动计算） |
| `qos` | QoS 等级（0/1/2） |
| `retain` | 保留标志 |

**返回值**：成功返回消息 ID，失败返回 -1

---

### `axk_mqtt_client_enqueue`

非阻塞发布（放入队列）。

```c
int axk_mqtt_client_enqueue(axk_mqtt_client_handle_t client,
                             const char *topic,
                             const char *data, int len,
                             int qos, int retain,
                             bool store);
```

---

### `axk_mqtt_client_destroy`

销毁客户端，释放资源。

```c
axk_err_t axk_mqtt_client_destroy(axk_mqtt_client_handle_t client);
```

---

### `axk_mqtt_set_config`

更新客户端配置。

```c
axk_err_t axk_mqtt_set_config(axk_mqtt_client_handle_t client,
                                const axk_mqtt_client_config_t *config);
```

---

### `axk_mqtt_client_register_event`

注册事件处理器（替代回调方式）。

```c
axk_err_t axk_mqtt_client_register_event(axk_mqtt_client_handle_t client,
                                           axk_mqtt_event_id_t event,
                                           axk_event_handler_t event_handler,
                                           void *event_handler_arg);
```

---

## 使用示例

### 基本 MQTT 连接（QoS 0）

```c
#include "mqtt_client.h"

static const char *mqtt_uri = "mqtt://mqtt.eclipse.org:1883";
static const char *sub_topic = "bl602/test";
static const char *pub_topic = "bl602/status";

static void mqtt_event_handler(void *handler_args,
                                esp_event_base_t base,
                                int32_t event_id,
                                void *event_data)
{
    axk_mqtt_event_t *event = (axk_mqtt_event_t *)event_data;

    switch (event->event_id) {
    case MQTT_EVENT_CONNECTED:
        printf("MQTT connected\r\n");
        // 订阅主题
        axk_mqtt_client_subscribe(event->client, sub_topic, 0);
        // 发布在线消息
        axk_mqtt_client_publish(event->client, pub_topic,
                                "online", 6, 0, 1);
        break;

    case MQTT_EVENT_DISCONNECTED:
        printf("MQTT disconnected\r\n");
        break;

    case MQTT_EVENT_SUBSCRIBED:
        printf("Subscribed to: %s\r\n", sub_topic);
        break;

    case MQTT_EVENT_DATA:
        printf("Message on %.*s: %.*s\r\n",
               event->topic_len, event->topic,
               event->data_len, event->data);
        break;

    case MQTT_EVENT_ERROR:
        printf("MQTT error\r\n");
        break;
    }
}

void mqtt_app_start(void)
{
    // 配置 MQTT 客户端
    axk_mqtt_client_config_t config = {
        .uri = mqtt_uri,
        .event_handle = mqtt_event_handler,
        .keepalive = 120,
        .disable_auto_reconnect = false,
    };

    // 初始化并启动
    axk_mqtt_client_handle_t client = axk_mqtt_client_init(&config);
    if (client) {
        axk_mqtt_client_start(client);
    }
}
```

### MQTT over SSL（双向认证）

```c
axk_mqtt_client_config_t ssl_config = {
    .uri = "mqtts://your-mqtt-server.com:8883",
    .event_handle = mqtt_event_handler,
    .cert_pem = ca_cert_pem,          // 服务器 CA 证书
    .client_cert_pem = client_cert,   // 客户端证书
    .client_key_pem = client_key,     // 客户端私钥
    .skip_cert_common_name_check = false,
};

axk_mqtt_client_handle_t ssl_client = axk_mqtt_client_init(&ssl_config);
axk_mqtt_client_start(ssl_client);
```

### MQTT over WebSocket

```c
axk_mqtt_client_config_t ws_config = {
    .uri = "wss://your-mqtt-server.com:443/mqtt",
    .event_handle = mqtt_event_handler,
    .transport = MQTT_TRANSPORT_OVER_WSS,
};

axk_mqtt_client_handle_t ws_client = axk_mqtt_client_init(&ws_config);
axk_mqtt_client_start(ws_client);
```

---

## 传输类型

```c
typedef enum {
    MQTT_TRANSPORT_UNKNOWN = 0x0,
    MQTT_TRANSPORT_OVER_TCP,      // mqtt://
    MQTT_TRANSPORT_OVER_SSL,      // mqtts://
    MQTT_TRANSPORT_OVER_WS,       // ws://
    MQTT_TRANSPORT_OVER_WSS       // wss://
} axk_mqtt_transport_t;
```

## MQTT 版本

```c
typedef enum {
    MQTT_PROTOCOL_UNDEFINED = 0,
    MQTT_PROTOCOL_V_3_1,      // MQTT 3.1
    MQTT_PROTOCOL_V_3_1_1     // MQTT 3.1.1（默认）
} axk_mqtt_protocol_ver_t;
```
