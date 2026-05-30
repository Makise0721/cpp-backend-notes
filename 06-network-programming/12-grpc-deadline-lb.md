# 02 — gRPC Deadline、超时控制与负载均衡

> 关联篇目：[11 调用模式与拦截器](./11-grpc-streaming-interceptor.md)

---

## 1. Deadline——RPC 调用的"死线"

**每个 gRPC 调用都应设置 deadline，防止无限等待。**

```cpp
grpc::ClientContext ctx;
ctx.set_deadline(std::chrono::system_clock::now() + std::chrono::seconds(5));
// 或
ctx.set_deadline(std::chrono::system_clock::now() + std::chrono::milliseconds(500));

auto status = stub_->Process(&ctx, request, &response);
if (status.error_code() == grpc::DEADLINE_EXCEEDED) {
    // 超时处理——降级、重试、返回默认值
}
```

### 1.1 Deadline 传播——防止级联超时

```
Service A (deadline=5s)
    │ 剩余 4.5s
    ▼
Service B (deadline=A 的剩余时间)
    │ 剩余 4s
    ▼
Service C (deadline=B 的剩余时间)

→ Service C 处理太慢 → 自动取消 → 避免了 A 已经超时但 C 还在浪费资源的"空中楼阁"
```

```cpp
// Server 端获取 Client 传来的 deadline
grpc::ServerContext* ctx;
auto deadline = ctx->deadline();  // 还剩多少时间
if (deadline - std::chrono::system_clock::now() < 100ms) {
    // 时间不够了，不调下游直接返回
    return grpc::Status(grpc::DEADLINE_EXCEEDED, "insufficient time");
}
```

### 1.2 Timeout vs Deadline

| | Timeout | Deadline |
|------|------|------|
| 含义 | 绝对等待时长 | 绝对时间点 |
| 传播 | ❌ | ✅（gRPC 自动传播到下游） |
| 设置 | `set_deadline(now + 5s)` 等价于 timeout=5s | `set_deadline(timestamp)` |

**永远用 Deadline，不要用纯 Timeout**。gRPC 自动把 deadline 传递到下游，形成一条截止时间链。

---

## 2. 负载均衡——客户端如何选择服务端

### 2.1 三种 LB 策略

```
1. 服务端 LB（传统）:
   Client → LB → Server1, Server2
   简单，但 LB 成瓶颈

2. 客户端 LB（gRPC 默认推荐）:
   Client ← 从注册中心获取 Server 列表
   Client → 自己选一个 Server 直连
   无中间跳，低延迟

3. Lookaside LB（Envoy/Linkerd）:
   Client → Sidecar → Server
   Sidecar 负责 LB + 重试 + 熔断
```

### 2.2 gRPC 内置 LB 策略

```cpp
// 设置 LB 策略
grpc::ChannelArguments args;
args.SetLoadBalancingPolicyName("round_robin");
auto channel = grpc::CreateCustomChannel("dns:///myservice:8080",
    grpc::InsecureChannelCredentials(), args);
```

| 策略 | 行为 |
|------|------|
| `pick_first` | 选第一个可用地址（默认） |
| `round_robin` | 轮询分发 |
| `grpclb` | 外部 LB 服务 |
| `xds` | Envoy xDS 动态配置 |

### 2.3 Name Resolution——如何从服务名找到 IP

```
"dns:///myservice:8080"
   │         │
   └─ scheme └─ 服务名

解析过程:
dns:///myservice:8080
  → DNS 解析 → [10.0.0.1, 10.0.0.2, 10.0.0.3]
  → round_robin → 选 10.0.0.1
  → 建立 HTTP/2 连接
```

在 K8s 中：
```
grpc:///user-svc.default.svc.cluster.local:50051
→ K8s DNS → Service ClusterIP → kube-proxy → Pod
或
dns:///user-svc-headless:50051
→ Headless Service → 直接返回 Pod IP 列表 → gRPC 客户端 LB
```

---

## 3. 超时与重试

```cpp
// Service Config (JSON) 配置重试策略
{
  "methodConfig": [{
    "name": [{"service": "myservice.MyService"}],
    "retryPolicy": {
      "maxAttempts": 3,
      "initialBackoff": "0.1s",
      "maxBackoff": "1s",
      "backoffMultiplier": 2,
      "retryableStatusCodes": ["UNAVAILABLE", "DEADLINE_EXCEEDED"]
    }
  }]
}
```

**重试前提**：操作必须幂等！非幂等操作只重试 `UNAVAILABLE`（确认服务端没有收到请求）。

---

## 小结

| 概念 | 一句话 |
|------|--------|
| Deadline | 绝对时间点，自动传播到下游链 |
| Deadline 传播 | 防止服务链中超时浪费 |
| 客户端 LB | gRPC 内置 round_robin/pick_first |
| Headless Service | K8s DNS 直接返回 Pod IP，配合客户端 LB |
| 重试 | 只对幂等操作和 UNAVAILABLE 重试 |

下一篇 [03 gRPC 生产实践：Keepalive、健康检查、Channel 管理](./03-grpc-production.md)。
