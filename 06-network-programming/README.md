# 第六章：网络编程与 gRPC

> 13 篇 · 从 TCP/IP 协议栈到 gRPC 生产实践，覆盖网络编程全链路。

### 网络协议与 IO (01-10)

| # | 核心内容 |
|---|------|
| 01 | OSI/TCP-IP 分层、IP/CIDR、ARP、ICMP、网络字节序 |
| 02 | UDP 特点、QUIC(0-RTT/多路复用/连接迁移) |
| 03 | 三次握手、四次挥手、TCP 状态机、TIME_WAIT |
| 04 | 滑动窗口、流量控制、拥塞控制(CUBIC/BBR) |
| 05 | 粘包、SO_REUSEADDR/REUSEPORT/NODELAY、DNS |
| 06 | HTTP/1.1 状态码、HTTPS(TLS 1.2/1.3)、HTTP/2、HTTP/3 |
| 07 | BIO/NIO、select/poll/epoll(LT/ET)、io_uring |
| 08 | Reactor(单/主从)、Proactor、muduo one-loop-per-thread |
| 09 | 零拷贝、缓冲区设计、协议设计、Protobuf、RPC、定时器 |
| 10 | tcpdump/ss/strace 网络排查、wrk/iperf 压测 |

### gRPC 深入 (11-13)

| # | 核心内容 |
|---|------|
| 11 | Unary/ServerStream/ClientStream/Bidi、拦截器链 |
| 12 | Deadline 传播、客户端 LB、重试策略 |
| 13 | Keepalive、健康检查、Channel 复用、反射调试 |

路线：协议栈→TCP→应用层→IO模型→高性能→gRPC
