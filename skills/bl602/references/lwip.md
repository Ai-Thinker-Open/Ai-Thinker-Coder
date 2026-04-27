# LwIP Socket API 参考

> 来源文件：`components/network/lwip/src/include/lwip/sockets.h`  
> LwIP 是轻量级 TCP/IP 协议栈，BL602 使用它提供标准 BSD Socket 接口。

---

## 概述

LwIP Socket API 兼容标准 POSIX socket 接口，支持 TCP、UDP、RAW IP 多种协议。BL602 默认配置支持 TCP/UDP，不支持 ICMP（ping 通过 `netutils/ping` 实现）。

---

## 常用头文件

```c
#include <lwip/sockets.h>
#include <lwip/netdb.h>
#include <lwip/errno.h>
```

---

## Socket 创建与销毁

### `socket`

创建 socket。

```c
int socket(int domain, int type, int protocol);
```

| 参数 | 说明 |
|------|------|
| `domain` | `AF_INET`（IPv4） |
| `type` | `SOCK_STREAM`（TCP）、`SOCK_DGRAM`（UDP）、`SOCK_RAW`（RAW） |
| `protocol` | `0`（自动选择） |

**返回值**：成功返回 socket 描述符，失败返回 `-1`

---

### `close`

关闭 socket。

```c
int close(int s);
```

---

## 地址绑定

### `bind`

绑定 IP 地址和端口。

```c
int bind(int s, const struct sockaddr *name, socklen_t namelen);
```

---

## TCP 连接

### `connect`

连接远程服务器（TCP）。

```c
int connect(int s, const struct sockaddr *name, socklen_t namelen);
```

---

### `listen`

监听端口（TCP 服务器）。

```c
int listen(int s, int backlog);
```

---

### `accept`

接受客户端连接。

```c
int accept(int s, struct sockaddr *addr, socklen_t *addrlen);
```

---

## 数据收发

### `send`

发送数据（TCP）。

```c
int send(int s, const void *data, size_t size, int flags);
```

---

### `recv`

接收数据（TCP）。

```c
int recv(int s, void *mem, size_t len, int flags);
```

**返回值**：成功返回字节数，`0` 表示对端关闭，`-1` 表示错误

---

### `sendto`

发送数据（UDP）。

```c
int sendto(int s, const void *data, size_t size, int flags,
           const struct sockaddr *to, socklen_t tolen);
```

---

### `recvfrom`

接收数据（UDP，可获取发送方地址）。

```c
int recvfrom(int s, void *mem, size_t len, int flags,
             struct sockaddr *from, socklen_t *fromlen);
```

---

## 读写

### `read`

读取数据（TCP/UDP）。

```c
int read(int s, void *buf, size_t len);
```

---

### `write`

写入数据（TCP/UDP）。

```c
int write(int s, const void *buf, size_t len);
```

---

## 关闭连接

### `shutdown`

关闭读写通道。

```c
int shutdown(int s, int how);
```

| `how` | 说明 |
|-------|------|
| `0` | 关闭读通道 |
| `1` | 关闭写通道 |
| `2` | 关闭读写通道 |

---

## 地址转换

### `inet_pton`

将字符串 IP 转换为二进制格式。

```c
int inet_pton(int af, const char *src, void *dst);
```

| `af` | 说明 |
|------|------|
| `AF_INET` | IPv4 |

---

### `inet_ntop`

将二进制 IP 转换为字符串格式。

```c
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
```

---

## 域名解析

### `gethostbyname`

通过域名获取 IP 地址。

```c
struct hostent *gethostbyname(const char *name);
```

**返回值**：`struct hostent *`，成功后 `h_addr_list[0]` 为 IP 地址

---

## 选项设置

### `setsockopt`

设置 socket 选项。

```c
int setsockopt(int s, int level, int optname, const void *optval, socklen_t optlen);
```

常用选项：

| level | optname | 说明 |
|-------|---------|------|
| `SOL_SOCKET` | `SO_KEEPALIVE` | TCP 保活 |
| `SOL_SOCKET` | `SO_RCVTIMEO` | 接收超时 |
| `SOL_SOCKET` | `SO_SNDTIMEO` | 发送超时 |
| `IPPROTO_TCP` | `TCP_NODELAY` | 禁用 Nagle 算法 |

---

## select 多路复用

### `select`

监控多个 socket 的可读/可写状态。

```c
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```

---

## 使用示例

### TCP Client

```c
#include <lwip/sockets.h>

int sock = socket(AF_INET, SOCK_STREAM, 0);
if (sock < 0) return -1;

struct sockaddr_in server;
server.sin_family = AF_INET;
server.sin_port = htons(8080);
inet_pton(AF_INET, "192.168.1.100", &server.sin_addr);

if (connect(sock, (struct sockaddr *)&server, sizeof(server)) < 0) {
    close(sock);
    return -1;
}

// 发送数据
const char *msg = "Hello\r\n";
send(sock, msg, strlen(msg), 0);

// 接收响应
char buf[256];
int len = recv(sock, buf, sizeof(buf) - 1, 0);
if (len > 0) {
    buf[len] = '\0';
    printf("Response: %s\r\n", buf);
}

close(sock);
```

### TCP Server

```c
int server_sock = socket(AF_INET, SOCK_STREAM, 0);

struct sockaddr_in local;
local.sin_family = AF_INET;
local.sin_port = htons(8080);
local.sin_addr.s_addr = INADDR_ANY;
bind(server_sock, (struct sockaddr *)&local, sizeof(local));

listen(server_sock, 5);

while (1) {
    struct sockaddr_in client;
    socklen_t len = sizeof(client);
    int client_sock = accept(server_sock, (struct sockaddr *)&client, &len);

    char buf[256];
    int n = recv(client_sock, buf, sizeof(buf), 0);
    if (n > 0) {
        send(client_sock, buf, n, 0); // echo 回去
    }
    close(client_sock);
}
```

### UDP Client

```c
int sock = socket(AF_INET, SOCK_DGRAM, 0);

struct sockaddr_in server;
server.sin_family = AF_INET;
server.sin_port = htons(8888);
inet_pton(AF_INET, "192.168.1.100", &server.sin_addr);

const char *msg = "UDP test\r\n";
sendto(sock, msg, strlen(msg), 0, (struct sockaddr *)&server, sizeof(server));

char buf[256];
struct sockaddr_in from;
socklen_t fromlen = sizeof(from);
int len = recvfrom(sock, buf, sizeof(buf), 0, (struct sockaddr *)&from, &fromlen);

close(sock);
```
