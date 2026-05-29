# 第八章：Linux 性能分析与调试

> 6 篇 · 从 perf 火焰图到网络抓包，覆盖 Linux 性能分析和故障排查全工具链。

---

## 篇目

| # | 文件 | 核心内容 |
|---|------|----------|
| 01 | [01-perf-flamegraph.md](./01-perf-flamegraph.md) | perf record/report/top/stat、火焰图生成与解读（宽顶=耗CPU多） |
| 02 | [02-system-monitoring.md](./02-system-monitoring.md) | top/htop、pidstat、vmstat、iostat、sar、free、/proc/meminfo |
| 03 | [03-gdb-core-dump.md](./03-gdb-core-dump.md) | GDB 断点/watchpoint、Core Dump 分析、bt full、thread apply all bt、多线程调试 |
| 04 | [04-strace-binary-analysis.md](./04-strace-binary-analysis.md) | strace 系统调用追踪、ltrace、ldd、nm、objdump、readelf、strings |
| 05 | [05-sanitizers-valgrind.md](./05-sanitizers-valgrind.md) | ASan/TSan/UBSan、Valgrind Memcheck、Heaptrack/Massif |
| 06 | [06-network-debugging.md](./06-network-debugging.md) | tcpdump/Wireshark、ss、dig/nslookup、curl -v、iftop/nethogs |

## 路线

性能分析(1-2)→调试(3-4)→内存检测(5)→网络排查(6)。先定位问题，再深入根因。
