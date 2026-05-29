# 06 — 网络调试工具：tcpdump、ss、dig、curl、iftop

> 关联篇目：[第一章 10 网络排查](../06-network-programming/10-troubleshooting.md) | [02 系统监控](./02-system-monitoring.md)

---

## 1. `tcpdump` + Wireshark——数据包级调试

### 1.1 抓包基础

```bash
# 抓指定端口
tcpdump -i eth0 port 8080 -w capture.pcap

# 抓指定主机 + 端口
tcpdump -i any host 192.168.1.100 and port 443 -w https.pcap

# 实时查看（不保存）
tcpdump -i eth0 -A port 80    # -A: ASCII 格式显示内容
tcpdump -i eth0 -X port 80    # -X: 同时显示 hex 和 ASCII

# 只抓 SYN 包（排查连接建立问题）
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'
```

### 1.2 精确过滤

```bash
# 抓取 HTTP GET 请求
tcpdump -i eth0 -A -s 0 'tcp port 80 and (tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420)'

# 抓取大于 MTU 的包（排查分片问题）
tcpdump -i eth0 'ip[6:2] & 0x1fff != 0'

# 统计重传包（排查丢包）
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn == 0 and tcp[tcpflags] & tcp-ack != 0' \
  | grep -c 'seq.*ack'
```

### 1.3 Wireshark 分析

```
常用 Wireshark 过滤器:

tcp.port == 8080           # 过滤端口
ip.addr == 192.168.1.100   # 过滤 IP
http.request               # 只显示 HTTP 请求
tcp.analysis.retransmission # 显示 TCP 重传
tcp.analysis.duplicate_ack  # 显示重复 ACK

Statistics → IO Graph       # 吞吐量曲线图
Statistics → Flow Graph     # TCP 流可视化
```

---

## 2. `ss`——Socket 统计（替代 netstat）

```bash
# 查看所有 TCP 监听端口
ss -tlnp

# 查看所有 TCP 连接（含状态）
ss -tan

# 按状态过滤
ss -tan state established
ss -tan state time-wait
ss -tan state syn-recv      # 看半连接队列情况

# 查看进程的 socket
ss -tnp | grep nginx

# 汇总统计
ss -s

# 查看 socket 详情（包括拥塞窗口、RTT 等）
ss -ti                    # TCP 内部信息（cwnd, rtt, rto, mss...）
ss -ti | grep -E 'cwnd|rtt|retrans'
```

---

## 3. `nslookup` / `dig`——DNS 排查

```bash
# 基本查询
nslookup example.com
dig example.com

# 指定 DNS 服务器
dig @8.8.8.8 example.com

# 反向查询（IP → 域名）
dig -x 8.8.8.8

# 查看完整解析链（+trace）
dig +trace example.com

# 只看简短结果
dig +short example.com

# 查特定类型记录
dig example.com A      # A 记录
dig example.com AAAA   # AAAA (IPv6)
dig example.com MX     # 邮件记录
dig example.com NS     # 域名服务器
dig example.com TXT    # TXT 记录
```

---

## 4. `curl -v`——HTTP 全链路调试

```bash
# 详细模式（显示请求头 + 响应头 + TLS 握手信息）
curl -v https://api.example.com/endpoint

# 输出:
*   Trying 1.2.3.4:443...
* Connected to api.example.com (1.2.3.4) port 443
* ALPN: offers h2,http/1.1
* TLSv1.3, TLS handshake, Client hello (1):
* TLSv1.3, TLS handshake, Server hello (2):
* TLSv1.3, TLS handshake, Encrypted Extensions (8):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
> GET /endpoint HTTP/1.1
> Host: api.example.com
> User-Agent: curl/8.0
>
< HTTP/1.1 200 OK
< Content-Type: application/json
< Content-Length: 1234

# 指定请求方法
curl -X POST -d '{"key":"value"}' -H "Content-Type: application/json" URL

# 显示耗时分解
curl -w "@curl-format.txt" -o /dev/null -s URL
# curl-format.txt:
# time_namelookup:  %{time_namelookup}\n
# time_connect:     %{time_connect}\n
# time_appconnect:  %{time_appconnect}\n
# time_starttransfer: %{time_starttransfer}\n
# time_total:       %{time_total}\n
```

---

## 5. `iftop` / `nethogs`——实时流量监控

### 5.1 `iftop`——按连接看流量

```bash
# 需要 root 权限
sudo iftop -i eth0

# 显示:
#  源IP           目的IP          2s    10s   40s(累计)
#  10.0.0.1  →  10.0.0.2      50Mb  200Mb  800Mb ← 谁在吃带宽？
#  10.0.0.3  →  10.0.0.4       1Mb    5Mb   20Mb

# 按端口过滤
sudo iftop -i eth0 -f "port 8080"
```

### 5.2 `nethogs`——按进程看流量

```bash
# 查看哪个进程在占用带宽
sudo nethogs eth0

# 输出:
# PID   USER   PROGRAM        DEV   SENT   RECEIVED
# 1234  root   nginx          eth0  100MB  500MB  ← 进程级别的流量！

# 指定接口
sudo nethogs eth0 wlan0
```

---

## 6. 网络调试工具箱

```
能不能连上？
├─ ping <host>                  # 基础可达性
├─ telnet <host> <port>         # TCP 端口可达性
├─ nc -zv <host> <port>         # 同上
└─ traceroute <host>            # 路由路径

DNS 是不是有问题？
├─ dig <domain>                 # DNS 解析
├─ dig @8.8.8.8 <domain>        # 指定 DNS
└─ dig +trace <domain>          # 完整解析链

TCP 连接怎么了？
├─ ss -tan                      # 连接状态
├─ tcpdump -i any port <port>   # 抓包
└─ curl -v <url>                # HTTP 级别调试

谁在吃带宽？
├─ iftop -i eth0                # 按连接
└─ nethogs eth0                 # 按进程

性能如何？
├─ curl -w "@fmt.txt" <url>     # HTTP 耗时分解
└─ wrk -t4 -c100 -d30s <url>    # 压测（见网络编程章）
```

---

## 第八章小结

| # | 篇目 | 核心 |
|---|------|------|
| 01 | perf 与火焰图 | perf record/report/top/stat + 火焰图解读 |
| 02 | 系统监控 | top/htop/pidstat/vmstat/iostat/sar/free |
| 03 | GDB 与 Core Dump | 断点/watchpoint/bt full/thread apply all bt |
| 04 | strace/二进制分析 | strace/ltrace/ldd/nm/objdump/strings |
| 05 | Sanitizer/Valgrind | ASan/TSan/UBSan/Valgrind Memcheck/heaptrack |
| 06 | 网络调试 | tcpdump/Wireshark/ss/dig/curl/iftop/nethogs |
