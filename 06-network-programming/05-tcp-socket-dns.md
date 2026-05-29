# 05 — TCP 粘包、Socket 选项、DNS、Nagle、SO_REUSEPORT

> 关联篇目：[03 TCP 握手](./03-tcp-handshake-state.md) | [04 拥塞控制](./04-tcp-flow-congestion.md)

---

## 1. TCP 粘包/分包——字节流的固有属性

TCP 是**字节流**协议，没有消息边界。`send("hello")` + `send("world")` 可能被接收为一次 `recv` → `"helloworld"`（粘包），也可能分成三次 `recv` → `"hel"` `"lo"` `"world"`（分包）。

### 1.1 三种解决方式

```cpp
// 方式 1：定长消息
struct Message { char data[1024]; };
// 每个消息固定大小，不足补齐

// 方式 2：分隔符
// 用特殊字符（如 \n、\r\n）分隔消息
// 例：HTTP 用 \r\n\r\n 分隔头部和体

// 方式 3：长度前缀（最常用）
// [4字节长度][N字节数据][4字节长度][N字节数据]...
void sendMessage(int fd, const std::string& data) {
    uint32_t len = htonl(data.size());
    send(fd, &len, 4, 0);
    send(fd, data.data(), data.size(), 0);
}
std::string recvMessage(int fd) {
    uint32_t len;
    recv(fd, &len, 4, MSG_WAITALL);
    len = ntohl(len);
    std::string data(len, '\0');
    recv(fd, &data[0], len, MSG_WAITALL);
    return data;
}
```

---

## 2. Socket 编程基础

### 2.1 标准流程

```
Server:                                Client:
socket()                               socket()
bind()                                    │
listen()                                  │
accept() ← 阻塞等待                       │
    │                                    │
    └──── 三次握手 ───────────────── connect()
    │                                    │
read/write ◄────────────────────────► read/write
    │                                    │
close()                                close()
```

### 2.2 关键函数

```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);  // AF_INET=IPv4, SOCK_STREAM=TCP

struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);
addr.sin_addr.s_addr = inet_addr("0.0.0.0");

bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));
listen(sockfd, 128);  // backlog = 全连接队列大小

int clientfd = accept(sockfd, NULL, NULL);
// 用 getaddrinfo 做 DNS 解析
struct addrinfo hints = {.ai_family = AF_INET, .ai_socktype = SOCK_STREAM};
struct addrinfo* res;
getaddrinfo("example.com", "80", &hints, &res);
int fd = socket(res->ai_family, res->ai_socktype, res->ai_protocol);
connect(fd, res->ai_addr, res->ai_addrlen);
```

---

## 3. 关键 Socket 选项

### 3.1 `SO_REUSEADDR`——地址复用

```c
int opt = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

- 允许 bind 到 TIME_WAIT 状态的端口
- **服务端必备**——否则重启后需等 2MSL（60s）才能重新 bind

### 3.2 `SO_REUSEPORT`——端口复用

```c
setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
```

- 多个进程/线程可以 bind **同一个 IP:Port**
- 内核自动将新连接分发给不同的 socket（负载均衡）
- **Nginx、Envoy 用它实现多 worker 监听同端口，无需代理分发**

### 3.3 `TCP_NODELAY`——禁用 Nagle 算法

```c
int opt = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));
```

Nagle 算法：小的 TCP 包会被延迟（攒到 MSS 或收到 ACK 再发），以减少网络中小包的数量。

| Nagle 开（默认） | Nagle 关（TCP_NODELAY） |
|------------------|------------------------|
| 减少小包，提高带宽利用率 | 立即发送，减少延迟 |
| 可能增加交互延迟 | 适合延迟敏感的场景（游戏、SSH） |

### 3.4 `SO_KEEPALIVE`——TCP 保活

```c
int opt = 1;
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &opt, sizeof(opt));
// 配合调整间隔
// sysctl net.ipv4.tcp_keepalive_time = 7200 (默认 2 小时，太长)
```

定期发送探测包确认对端存活，超时后关闭连接。

---

## 4. DNS 解析——`getaddrinfo`

```cpp
#include <netdb.h>

// 替代过时的 gethostbyname
struct addrinfo hints = {};
hints.ai_family = AF_UNSPEC;     // 支持 IPv4/6
hints.ai_socktype = SOCK_STREAM;

struct addrinfo* result;
int ret = getaddrinfo("www.example.com", "80", &hints, &result);
if (ret != 0) {
    fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(ret));
    return;
}

// 遍历结果（可能返回多个 IP）
for (struct addrinfo* rp = result; rp != NULL; rp = rp->ai_next) {
    int fd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
    if (connect(fd, rp->ai_addr, rp->ai_addrlen) == 0)
        break;  // 成功
    close(fd);
}
freeaddrinfo(result);
```

---

## 5. `SO_LINGER`——控制 close 行为

```c
struct linger l;
l.l_onoff = 1;   // 启用
l.l_linger = 5;  // 等待 5 秒

// 默认行为（l_onoff=0）：close 立即返回，内核后台发送剩余数据
// l_onoff=1, l_linger=0：直接发 RST，不经过四次挥手（数据可能丢失）
// l_onoff=1, l_linger>0：close 阻塞最多 linger 秒，尽量发送完剩余数据
setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 粘包 | TCP 字节流无边界 → 定长/分隔符/长度前缀 |
| `SO_REUSEADDR` | 允许 bind TIME_WAIT 端口（服务端必须） |
| `SO_REUSEPORT` | 多进程 bind 同端口，内核分发 |
| `TCP_NODELAY` | 禁用 Nagle，低延迟优先 |
| Nagle | 攒小包，减少网络开销 |
| `SO_KEEPALIVE` | TCP 保活探测 |
| `getaddrinfo` | DNS 解析，替代 `gethostbyname` |

下一篇 [06 HTTP/1.1、HTTPS/TLS、HTTP/2、HTTP/3](./06-http-tls-quic.md)。
