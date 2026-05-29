# 第六章：网络编程

> 10 篇 · 从 TCP/IP 协议栈到高性能网络架构，覆盖网络编程全链路。

---

## 篇目

| # | 文件 | 核心内容 |
|---|------|----------|
| 01 | [01-network-layers-ip-arp.md](./01-network-layers-ip-arp.md) | OSI/TCP-IP 分层、IP/CIDR、ARP、ICMP(ping/traceroute/TTL)、网络字节序 |
| 02 | [02-udp-quic.md](./02-udp-quic.md) | UDP 特点与适用场景、QUIC(0-RTT/多路复用无阻塞/连接迁移) |
| 03 | [03-tcp-handshake-state.md](./03-tcp-handshake-state.md) | 三次握手(为什么三次)、四次挥手(为什么四次)、TCP 状态机、TIME_WAIT |
| 04 | [04-tcp-flow-congestion.md](./04-tcp-flow-congestion.md) | 滑动窗口、流量控制(RWND)、拥塞控制(慢启动/快重传/快恢复)、CUBIC/BBR |
| 05 | [05-tcp-socket-dns.md](./05-tcp-socket-dns.md) | TCP 粘包解决方案、SO_REUSEADDR/REUSEPORT/NODELAY、DNS(getaddrinfo) |
| 06 | [06-http-tls-quic.md](./06-http-tls-quic.md) | HTTP/1.1 状态码、HTTPS(TLS 1.2/1.3 握手)、HTTP/2(多路复用/HPACK)、HTTP/3 |
| 07 | [07-io-models.md](./07-io-models.md) | BIO/NIO、select/poll/epoll(LT/ET)、io_uring |
| 08 | [08-reactor-proactor.md](./08-reactor-proactor.md) | Reactor(单/主从)、Proactor、muduo one-loop-per-thread |
| 09 | [09-high-performance.md](./09-high-performance.md) | 零拷贝(sendfile/splice)、缓冲区设计、协议设计、Protobuf、RPC、定时器(时间轮) |
| 10 | [10-troubleshooting.md](./10-troubleshooting.md) | tcpdump/ss/strace 网络排查、wrk/iperf 压测 |

## 路线

协议栈(1-2)→TCP 深入(3-5)→应用层(6)→IO 模型(7-8)→高性能实践(9)→排查工具(10)。
