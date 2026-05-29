# 09 — 零拷贝、协议设计、Protobuf、RPC、定时器

> 关联篇目：[07 IO 模型](./07-io-models.md) | [08 Reactor](./08-reactor-proactor.md)

---

## 1. 零拷贝——让数据不经过用户态

### 1.1 传统文件传输（4 次拷贝！）

```
传统 sendfile（read + write）:
磁盘 → (DMA) → 内核 buffer → (CPU) → 用户 buffer → (CPU) → socket buffer → (DMA) → 网卡
        1             2             3              4
```

### 1.2 `sendfile`——零拷贝

```c
// 直接把文件数据从内核 buffer 发到 socket，不经过用户态
sendfile(sockfd, filefd, &offset, count);
```

```
sendfile（2 次拷贝 + 2 次 DMA）:
磁盘 → (DMA) → 内核 buffer → (DMA) → 网卡
        1             2

如果网卡支持 scatter-gather:
磁盘 → (DMA) → 内核 buffer ─┐
                            ├→ 网卡 (DMA gather)
        内核只传描述符 ──────┘
→ 只有 1 次 DMA 拷贝 + 1 次 DMA gather！
```

### 1.3 `mmap`——用户态共享内核页

```c
void* ptr = mmap(NULL, size, PROT_READ, MAP_SHARED, fd, 0);
// 用户态直接读/写内核 page cache 中的数据，不拷贝
// write(sockfd, ptr, size); 仍然有一次 CPU 拷贝
```

### 1.4 `splice`——两个 fd 之间的管道传输

```c
// 在两个 fd 之间直接搬数据，不经过用户态
splice(filefd, &off, pipefd[1], NULL, len, SPLICE_F_MOVE);
splice(pipefd[0], NULL, sockfd, NULL, len, SPLICE_F_MOVE);
```

### 1.5 零拷贝对比

| 方案 | 数据经过用户态 | CPU 拷贝次数 | 适用 |
|------|:---:|:---:|------|
| read+write | ✅ | 2 | 需要修改数据 |
| `mmap` + write | ❌ | 1 | 大文件共享 |
| `sendfile` | ❌ | 0-1 | 静态文件服务 |
| `splice` | ❌ | 0 | 任意 fd 间传输 |
| `io_uring` | ❌ | 0 | 新一代通用方案 |

---

## 2. 缓冲区设计

### 2.1 读写缓冲区分离

```cpp
class Buffer {
    std::vector<char> buffer_;
    size_t readIdx_ = 0;   // 已读位置
    size_t writeIdx_ = 0;  // 已写位置

    // 可读区域: [readIdx_, writeIdx_)
    // 可写区域: [writeIdx_, buffer_.size())
    // 已读区域: [0, readIdx_) — 可腾挪

    void ensureWritable(size_t len) {
        if (writableBytes() < len) {
            // 先尝试腾挪已读空间
            if (readIdx_ > 0) {
                size_t readable = readableBytes();
                memmove(&buffer_[0], &buffer_[readIdx_], readable);
                readIdx_ = 0;
                writeIdx_ = readable;
            }
            if (writableBytes() < len)
                buffer_.resize(writeIdx_ + len);
        }
    }
};
```

### 2.2 环形缓冲区（Ring Buffer）

无锁 SPSC 队列的变体——适合单生产者单消费者：

```cpp
// 读写指针分别在 buffer 上绕圈
// readPos: 下一个可读位置
// writePos: 下一个可写位置
// 满: (writePos + 1) % size == readPos（牺牲一格）
```

---

## 3. 协议设计与私有通信协议

### 3.1 数据包基本结构

```
┌────────┬────────┬──────────┬────────┬──────────┬────────┐
│ 魔数    │ 版本号  │  消息类型  │ 长度   │  载荷    │ CRC校验 │
│ 4字节   │ 2字节   │  2字节    │ 4字节  │  N字节   │ 4字节   │
└────────┴────────┴──────────┴────────┴──────────┴────────┘

魔数: 固定值，用于快速识别协议（如 0xCAFEBABE）
版本号: 用于协议升级兼容
消息类型: 区分不同请求/响应
长度: 载荷长度（粘包解决方案）
CRC: 完整性校验
```

---

## 4. 序列化——Protobuf、JSON、MessagePack

### 4.1 对比

| | JSON | Protobuf | MessagePack |
|------|:---:|:---:|:---:|
| 可读性 | ✅ | ❌（二进制） | ❌（二进制） |
| 大小 | 大 | **极小** | 小 |
| 序列化速度 | 慢 | **快** | 中 |
| Schema | 无 | 有（`.proto` 文件） | 无 |
| 适用 | Web API、配置 | **RPC、高性能通信** | 存储 |

### 4.2 Protobuf 的变长编码

```
Varint: 每个字节的 MSB (最高位) 表示"后面还有没有更多字节"

数字 1:   0000 0001  → 1 字节
数字 300: 1010 1100  0000 0010  → 2 字节
          └─MSB=1──┘ └─MSB=0──┘（结束）

小数字用更少的字节
```

---

## 5. RPC 基础

**RPC** = Remote Procedure Call。像调用本地函数一样调用远程函数。

```
Client                              Server
  │                                   │
  │ stub.add(3,4)                     │
  │   ↓ 序列化                       │
  │ 网络发送 ──── [add, 3, 4] ────→   │
  │                                   │  反序列化 → add(3,4) → return 7
  │ ←─── [result: 7] ──── 网络回复   │
  │   ↑ 反序列化                     │
  │ return 7                          │
```

RPC 框架的核心组件：

```
┌──────────────────────────────────┐
│ IDL (接口定义语言)  → .proto / .thrift  │
├──────────────────────────────────┤
│ 代码生成 (Stub/Skeleton)        │
├──────────────────────────────────┤
│ 序列化层 (Protobuf/Thrift/...)   │
├──────────────────────────────────┤
│ 传输层 (TCP/UDP/HTTP)           │
├──────────────────────────────────┤
│ 服务发现 (etcd/consul/zookeeper) │
└──────────────────────────────────┘
```

---

## 6. 定时器设计

### 6.1 时间轮（Time Wheel）

```
时间轮（多层级，类似钟表）:

Level 0 (精度 1ms, 256 槽):  0ms ~ 255ms
Level 1 (精度 256ms, 64 槽): 256ms ~ 16s
Level 2 (精度 16s, 64 槽):   16s ~ 1024s

添加定时器: O(1) — 直接放到对应的槽
执行: 指针转动，到某槽时执行槽内所有定时器
```

**Linux 内核和 Netty 都用时间轮管理大量定时器**。

### 6.2 最小堆定时器

所有定时器按到期时间放在小顶堆里。`top()` = 最近到期的。

- 添加：O(log n)
- 取最小：O(1)
- 适合定时器数量不多 + 需要精确到期的场景

### 6.3 对比

| 方案 | 添加 | 删除 | 执行 | 大量定时器 |
|------|:---:|:---:|:---:|:---:|
| 时间轮 | O(1) | O(1) | O(1)/tick | ✅ 极好 |
| 最小堆 | O(log n) | O(log n) | O(1) | 中等 |
| 红黑树 | O(log n) | O(log n) | O(log n) | 中等 |

---

## 小结

| 概念 | 一句话 |
|------|--------|
| 零拷贝 | `sendfile`/`splice`/`mmap` 避免用户态数据拷贝 |
| 长度前缀 | 解决粘包的最常用方式 |
| Protobuf | Varint + ZigZag 编码，二进制序列化 |
| RPC | 远程调用框架 = IDL + 序列化 + 传输 + 服务发现 |
| 时间轮 | O(1) 添加/删除，适合海量定时器 |

下一篇 [10 网络分析与排查](./10-troubleshooting.md)。
