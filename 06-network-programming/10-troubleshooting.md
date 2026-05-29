# 10 — 网络分析与排查：tcpdump、netstat/ss、strace、压测

> 关联篇目：[03 TCP 状态机](./03-tcp-handshake-state.md) | [05 Socket 选项](./05-tcp-socket-dns.md)

---

## 1. `tcpdump` / `tshark`——抓包

### 1.1 常用过滤

```bash
# 抓某个端口
tcpdump -i eth0 port 80

# 抓某个主机
tcpdump -i eth0 host 192.168.1.100

# 抓 TCP SYN 包
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'

# 组合过滤
tcpdump -i eth0 'host 10.0.0.1 and port 443'

# 保存到文件（用 Wireshark 分析）
tcpdump -i eth0 -w capture.pcap

# 读取 pcap 文件
tcpdump -r capture.pcap

# tshark（tshark 是 Wireshark 的命令行版）
tshark -i eth0 -Y "http.request"
```

### 1.2 常用过滤器语法

| 过滤器 | 含义 |
|--------|------|
| `host 1.2.3.4` | 匹配 IP |
| `port 80` | 匹配端口 |
| `tcp` / `udp` | 匹配协议 |
| `src` / `dst` | 源/目的 |
| `net 192.168.1.0/24` | 匹配网段 |
| `tcp[tcpflags] & tcp-syn != 0` | SYN 包 |

---

## 2. `netstat` / `ss`——连接状态统计

```bash
# 查看所有 TCP 连接
ss -tlnp        # -t=tcp, -l=listen, -n=数字, -p=进程
ss -tan         # -a=所有（含 LISTEN 和 ESTABLISHED）

# 按状态统计
ss -s           # 总体统计
ss -tan state time-wait | wc -l   # TIME_WAIT 数量
ss -tan state established | wc -l  # ESTABLISHED 数量

# 查看某个进程的连接
ss -tnp | grep nginx

# netstat 等效命令（更老）
netstat -tlnp
netstat -s     # 协议统计（SYN 丢包、重传统计等）
```

---

## 3. `ping` / `traceroute` / `mtr`——连通性排查

```bash
# ping: 测试可达性 + RTT
ping -c 10 google.com       # 发 10 个包
ping -s 1400 google.com     # 指定包大小

# traceroute: 逐跳追踪路由
traceroute google.com       # 显示每一跳的 IP 和延迟

# mtr: ping + traceroute 组合（实时更新）
mtr google.com              # 动态显示每跳丢包率和延迟
```

---

## 4. 常见故障排查思路

### 4.1 TCP 连接不上

```
排查清单:
1. 服务端是否在监听？ → ss -tlnp
2. 防火墙是否允许？ → iptables -L -n
3. 客户端能否 ping 通？ → ping server_ip
4. 端口可 reachable？ → telnet server_ip port 或 nc -zv server_ip port
5. 半连接队列是否满？ → netstat -s | grep -i listen
6. 全连接队列是否满？ → ss -lnt (Recv-Q > Send-Q 表示积压)
7. 服务端是否 accept 太慢？ → strace -p <pid> 看 accept 调用频率
```

### 4.2 丢包排查

```bash
# 网卡错误统计
ethtool -S eth0 | grep -i error
ip -s link show eth0       # errors / dropped

# TCP 重传统计
netstat -s | grep retrans
ss -ti                     # 查看单连接的 retrans 计数

# 系统缓冲区
sysctl net.core.rmem_default
sysctl net.core.wmem_default
netstat -s | grep "packet receive errors"
```

### 4.3 延迟高

```
排查清单:
1. 网络 RTT → ping 看基础延迟
2. Nagle 算法？ → 检查是否设置了 TCP_NODELAY
3. 应用层慢？ → strace -T -p <pid> 看系统调用耗时
4. 重传多？ → ss -ti 看 retrans
5. TCP 缓冲不够？ → 增大 rmem/wmem
```

---

## 5. `strace`——追踪系统调用

```bash
# 追踪某个运行中的进程
strace -p <pid>

# 只看网络相关系统调用
strace -p <pid> -e trace=network

# 显示每个系统调用的耗时
strace -p <pid> -T

# 统计各系统调用的次数和耗时
strace -p <pid> -c

# 追踪新启动的进程
strace -f ./my_server   # -f 追踪子进程
```

---

## 6. 网络性能压测

### 6.1 HTTP 压测

```bash
# wrk —— 高性能 HTTP 压测
wrk -t4 -c100 -d30s http://localhost:8080/
#    4线程 100并发 持续30秒

# ab (Apache Bench)
ab -n 10000 -c 100 http://localhost:8080/

# vegeta —— Go 编写的压测工具
echo "GET http://localhost:8080/" | vegeta attack -duration=30s | vegeta report
```

### 6.2 带宽测试

```bash
# iperf3
# 服务端
iperf3 -s

# 客户端（测试 TCP 带宽）
iperf3 -c server_ip -t 30

# 测试 UDP
iperf3 -c server_ip -u -b 1G
```

---

## 7. 排查工具速查

| 工具 | 用途 | 常用命令 |
|------|------|----------|
| `tcpdump` | 抓包分析 | `tcpdump -i eth0 port 80 -w out.pcap` |
| `ss` | 连接状态 | `ss -tlnp` / `ss -tan state time-wait` |
| `ping` | 可达性 | `ping -c 10 host` |
| `traceroute` | 路由追踪 | `traceroute host` |
| `strace` | 系统调用追踪 | `strace -p PID -e trace=network` |
| `netstat -s` | 协议统计 | 丢包、重传、队列溢出 |
| `wrk` | HTTP 压测 | `wrk -t4 -c100 -d30s URL` |
| `iperf3` | 带宽测试 | `iperf3 -c host -t 30` |

---

## 第六章小结

| # | 篇目 | 核心 |
|---|------|------|
| 01 | 分层与 IP | OSI/TCP-IP、CIDR、ARP、ICMP、字节序 |
| 02 | UDP/QUIC | 无连接特点、QUIC 0-RTT/多路复用/连接迁移 |
| 03 | TCP 握手 | 三次握手/四次挥手/状态机/TIME_WAIT |
| 04 | 流量与拥塞 | 滑动窗口/cwnd/慢启动/快重传/CUBIC/BBR |
| 05 | Socket 与选项 | 粘包/REUSEADDR/REUSEPORT/NODELAY/DNS |
| 06 | 应用层协议 | HTTP/1.1 状态码/HTTPS TLS 1.3/HTTP2 多路复用/HTTP3 |
| 07 | IO 模型 | select/poll/epoll(LT/ET)/io_uring |
| 08 | Reactor/Proactor | 单/主从 Reactor、Proactor、muduo one-loop-per-thread |
| 09 | 高性能网络 | 零拷贝/缓冲区/协议设计/Protobuf/RPC/定时器 |
| 10 | 网络排查 | tcpdump/ss/strace/wrk/iperf |
