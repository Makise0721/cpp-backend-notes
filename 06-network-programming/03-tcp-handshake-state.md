# 03 — TCP 三次握手、四次挥手、状态机与 TIME_WAIT

> 关联篇目：[04 流量与拥塞控制](./04-tcp-flow-congestion.md) | [05 粘包与 socket](./05-tcp-socket-dns.md)

---

## 1. 三次握手——为什么是三次？

### 1.1 握手过程

```
Client                          Server
  │                               │
  │ ──── SYN, seq=x ───────────→  │  (1) SYN_SENT
  │                               │
  │ ←── SYN+ACK, seq=y, ack=x+1── │  (2) SYN_RCVD
  │                               │
  │ ──── ACK, seq=x+1, ack=y+1──→ │  (3) ESTABLISHED
  │                               │
  │         ESTABLISHED           │
```

**为什么不是两次？** 防止旧的重复连接请求到达服务器导致错误连接。

```
场景：Client 发了一个 SYN（seq=90），因网络延迟没到
Client 超时重发 SYN（seq=100）→ Server ACK → 连接建立
后来旧的 SYN（seq=90）终于到了

如果两次握手：Server 收到旧 SYN → 回复 SYN-ACK → 直接 ESTABLISHED
               但 Client 早就不要这个连接了 → Server 白等了

如果三次握手：Client 收到旧 SYN 的 ACK → 发现 seq 不对 → 发 RST 拒绝
```

**为什么不是四次？** 第三次握手的 ACK 可以与数据合并发送。服务端收到 ACK 后立即进入 ESTABLISHED。

### 1.2 TCP 头部关键字段

```
TCP 头部（20 字节基础）:
┌───────────────┬───────────────┐
│   源端口 16    │  目的端口 16   │
├───────────────┴───────────────┤
│          序列号 seq 32         │
├───────────────────────────────┤
│         确认号 ack 32          │
├───┬───┬───┬───┬───────────────┤
│数据│保留│UAPRSF│   窗口大小 16  │
│偏移│   │ ││││││               │
└───┴───┴─┴┴┴┴┴┴───────────────┘

标志位: URG ACK PSH RST SYN FIN
```

---

## 2. 四次挥手——为什么是四次？

### 2.1 挥手过程

```
Client                          Server
  │                               │
  │ ──── FIN, seq=u ───────────→  │  (1) FIN_WAIT_1
  │                               │
  │ ←── ACK, ack=u+1 ─────────── │  (2) CLOSE_WAIT
  │    (Client→Server 方向关闭)    │
  │                               │    应用可能还在发数据...
  │                               │
  │ ←── FIN, seq=v, ack=u+1 ──── │  (3) LAST_ACK
  │                               │
  │ ──── ACK, ack=v+1 ─────────→  │  (4) TIME_WAIT (2MSL)
  │                               │
  │          CLOSED               │
```

**为什么是四次？** TCP 是全双工的——每个方向需要单独关闭。收到对方的 FIN 只表示对方不再发数据，但己方可能还有数据要发。所以 ACK 和 FIN 分开发。

---

## 3. TCP 状态机

```
                              CLOSED
                                │
                    ┌── 被动打开 │ 主动打开 ──┐
                    ▼           │           ▼
                  LISTEN        │        SYN_SENT
                    │           │           │
                    └─ 收到 SYN ─┤           │
                    ▼           │           │
                 SYN_RCVD       │           │
                    │           │           │
                    └── 收到 ACK ─┴─────────┘
                                │
                                ▼
                          ESTABLISHED
                     ┌──────────┴──────────┐
                     │ 主动关闭   被动关闭   │
                     ▼                     ▼
                FIN_WAIT_1             CLOSE_WAIT
                     │                     │
                     ▼                     ▼
                FIN_WAIT_2              LAST_ACK
                     │                     │
                     ▼                     ▼
                TIME_WAIT               CLOSED
                 (2MSL)
                     │
                     ▼
                  CLOSED
```

---

## 4. TIME_WAIT——为什么需要 2MSL？

### 4.1 作用

1. **确保最后一个 ACK 能重传**：如果 Server 没收到 ACK，会重发 FIN，Client 需要 TIME_WAIT 来处理重传
2. **让旧连接的所有报文从网络中消失**：2MSL 之后，该连接期间的所有报文肯定都消亡了，不会干扰新连接

### 4.2 TIME_WAIT 的危害

如果服务器主动关闭连接（如 HTTP 服务端主动断开 Keep-Alive），会产生大量 TIME_WAIT 状态的 socket → 端口资源耗尽。

```bash
$ ss -tan | grep TIME_WAIT | wc -l
2847  # 大量 TIME_WAIT

# 每个 TIME_WAIT 占用一个本地端口 60 秒（2MSL=60s on Linux）
# 65535 个端口 ÷ 60s ≈ 最多 1092 连接/秒（不够用！）
```

### 4.3 优化措施

```bash
# 1. 开启 TIME_WAIT 复用（允许将 TIME_WAIT socket 用于新连接）
sysctl net.ipv4.tcp_tw_reuse=1

# 2. SO_REUSEADDR（应用层设置，允许 bind 到 TIME_WAIT 的端口）
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

# 3. 调整 TIME_WAIT 时长（慎用）
# net.ipv4.tcp_fin_timeout  # 控制 FIN_WAIT_2 时长，不是 TIME_WAIT

# 4. 减少服务端主动关闭（让客户端关，或 HTTP Keep-Alive）
```

---

## 5. 半关闭与半连接

### 5.1 `shutdown` vs `close`

```c
close(fd);     // 关闭 fd，引用计数减 1，到 0 时才真正关闭
shutdown(fd, SHUT_RD);   // 关闭读端
shutdown(fd, SHUT_WR);   // 关闭写端（发送 FIN）
shutdown(fd, SHUT_RDWR); // 关闭读写
```

`shutdown` 直接操作 TCP 连接（不管引用计数），常用于半关闭场景：读完请求后 `shutdown(SHUT_WR)` 发 FIN，但保持读端可收响应。

### 5.2 半连接队列与全连接队列

```
SYN 队列（半连接）：收到 SYN 但还没完成三次握手的连接
ACCEPT 队列（全连接）：已完成三次握手但还没被 accept() 取走的连接

队列满了 → 新连接被丢弃 → 客户端看到 "Connection refused"
```

```bash
# 查看队列溢出
$ netstat -s | grep -i listen
$ ss -lnt  # Recv-Q = 全连接队列积压数
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 三次握手 | SYN → SYN-ACK → ACK，防止旧连接干扰 |
| 四次挥手 | 全双工双向关闭，ACK 和 FIN 分开发 |
| TIME_WAIT | 等 2MSL 保证最后一个 ACK 可达 + 清理旧报文 |
| `shutdown` vs `close` | shutdown 不依赖引用计数，可半关闭 |
| 半/全连接队列 | 半连接 = SYN_RCVD，全连接 = ESTABLISHED 待 accept |

下一篇 [04 滑动窗口、流量控制与拥塞控制](./04-tcp-flow-congestion.md)。
