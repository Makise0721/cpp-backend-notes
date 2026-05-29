# 06 — HTTP/1.1、HTTPS/TLS、HTTP/2、HTTP/3

> 关联篇目：[02 UDP/QUIC](./02-udp-quic.md) | [03 TCP 握手](./03-tcp-handshake-state.md)

---

## 1. HTTP/1.1——Web 的基石

### 1.1 请求与响应格式

```
请求:
GET /index.html HTTP/1.1\r\n
Host: www.example.com\r\n
Accept: text/html\r\n
\r\n

响应:
HTTP/1.1 200 OK\r\n
Content-Type: text/html\r\n
Content-Length: 1234\r\n
\r\n
<html>...</html>
```

### 1.2 状态码

| 范围 | 含义 | 典型 |
|------|------|------|
| **1xx** | 信息 | 101 Switching Protocols |
| **2xx** | 成功 | 200 OK, 201 Created, 204 No Content |
| **3xx** | 重定向 | 301 永久, 302 临时, 304 Not Modified |
| **4xx** | 客户端错误 | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| **5xx** | 服务端错误 | 500 Internal Error, 502 Bad Gateway, 503 Unavailable, 504 Timeout |

### 1.3 长连接（Keep-Alive）

```
HTTP/1.0: 每次请求/响应 → 关闭连接 → 下次重新 TCP 握手
HTTP/1.1: 默认 Keep-Alive → 同一连接发送多个请求

Connection: keep-alive    ← 启用
Connection: close         ← 本次响应后关闭
```

### 1.4 管线化（Pipelining）

```
非管线化:
C → req1 → S → resp1 → C → req2 → S → resp2 → C

管线化:
C → req1 → C → req2 → S → resp1 → S → resp2 → C
（不等第一个响应就发第二个请求）

⚠️ 队头阻塞：resp1 处理慢了，resp2 即使好了也不能发
→ HTTP/2 用多路复用解决
```

---

## 2. HTTPS / TLS——加密的 HTTP

### 2.1 TLS 1.2 握手（2-RTT）

```
Client                                    Server
  │                                         │
  │ ── ClientHello ──────────────────────→  │ (支持的加密套件、随机数)
  │                     (1-RTT)             │
  │ ←── ServerHello + 证书 + ServerKeyExchange │
  │                                         │
  │ ── ClientKeyExchange + Finished ────→  │ (2-RTT)
  │                                         │
  │ ←── Finished ────────────────────────  │
  │                                         │
  │        加密通信                           │
```

### 2.2 TLS 1.3 握手（1-RTT）

```
Client                                          Server
  │                                               │
  │ ── ClientHello + 密钥交换参数 ──────────────→  │
  │                                               │  (1-RTT)
  │ ←── ServerHello + 密钥交换 + 加密的 Finished ── │
  │                                               │
  │ ── 加密的 Finished ────────────────────────→  │
  │                                               │
  │              加密通信                           │
```

**TLS 1.3 的改进**：
- 去掉 RSA 密钥交换，只用 ECDHE（前向安全性）
- 握手从 2-RTT 降到 1-RTT（加上 TCP 握手总共 2-RTT）
- 支持 0-RTT 恢复（PSK）

---

## 3. HTTP/2——多路复用与头部压缩

### 3.1 核心改进

```
HTTP/1.1: 一个 TCP 连接同时只能处理一个请求（串行）
HTTP/2:   一个 TCP 连接上同时传输多个流（多路复用）

┌─────────────────────────────────────┐
│           TCP 连接                    │
│  ┌── Stream 1: reqA → [A1][A2]     │
│  ├── Stream 2: reqB → [B1][B2][B3] │
│  ├── Stream 3: reqC → [C1]         │
│  └── Stream 4: Server Push         │
└─────────────────────────────────────┘

帧可以交织发送！
```

### 3.2 主要特性

| 特性 | 说明 |
|------|------|
| **二进制帧** | 不再是文本协议，解析更快 |
| **多路复用** | 一个连接多个流，帧可交织 |
| **头部压缩（HPACK）** | 静态/动态表，只发变更部分 |
| **Server Push** | 服务器主动推送资源（CSS/JS） |
| **流优先级** | 可以指定哪个流更重要 |

### 3.3 HTTP/2 的队头阻塞

HTTP/2 虽然解决了 HTTP 层面的队头阻塞，但底层仍是 TCP → **TCP 丢包会阻塞所有流**。HTTP/3 用 QUIC 解决。

---

## 4. HTTP/3 = QUIC + HTTP

```
HTTP/3 协议栈:
┌──────────────┐
│   HTTP/3     │ ← 基于 QUIC 的 HTTP
├──────────────┤
│    QUIC      │ ← 可靠传输 + TLS
├──────────────┤
│    UDP       │
├──────────────┤
│     IP       │
└──────────────┘
```

| | HTTP/2 | HTTP/3 (QUIC) |
|------|:---:|:---:|
| 传输 | TCP | UDP |
| 握手 | TCP(1RTT) + TLS(1RTT) = 2RTT | QUIC + TLS = 1RTT (0RTT 恢复) |
| 队头阻塞 | TCP 层有 | **无**（每个流独立） |
| 连接迁移 | ❌ | ✅（Connection ID） |

---

## 5. Cookie 与 Session

```
Cookie 流程:
1. Server: Set-Cookie: session_id=abc123
2. Browser 保存 cookie
3. 后续请求自动带上: Cookie: session_id=abc123
4. Server 根据 session_id 查 Session 存储（内存/Redis/DB）→ 获取用户状态

属性:
HttpOnly → JS 无法读取（防 XSS）
Secure   → 只在 HTTPS 下传输
SameSite → 防 CSRF
```

---

## 小结

| 协议 | 核心 | 问题 |
|------|------|------|
| HTTP/1.1 | 文本协议、Keep-Alive | 串行请求、队头阻塞 |
| HTTPS/TLS | 加密 + 身份认证 | 额外握手 RTT |
| TLS 1.3 | 1-RTT 握手、0-RTT 恢复 | — |
| HTTP/2 | 二进制帧、多路复用、HPACK | TCP 层队头阻塞 |
| HTTP/3 | 基于 QUIC | UDP 部署受限 |

下一篇 [07 IO 模型：select/poll/epoll/io_uring](./07-io-models.md)。
