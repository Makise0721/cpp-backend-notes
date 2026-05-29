# 03 — gRPC 生产实践：Keepalive、健康检查、Channel 管理

> 关联篇目：[01 调用模式](./01-grpc-streaming-interceptor.md) | [02 Deadline](./02-grpc-deadline-lb.md)

---

## 1. Keepalive——保持连接健康

HTTP/2 连接可能被中间代理/防火墙/负载均衡器静默关闭。Keepalive 定期发送 PING 帧探测连接。

### 1.1 服务端配置

```cpp
grpc::ServerBuilder builder;
builder.AddListeningPort("0.0.0.0:50051", grpc::InsecureServerCredentials());

// Keepalive 配置
builder.AddChannelArgument(GRPC_ARG_KEEPALIVE_TIME_MS, 30000);      // 每 30s 发 PING
builder.AddChannelArgument(GRPC_ARG_KEEPALIVE_TIMEOUT_MS, 10000);   // PING 超时 10s
builder.AddChannelArgument(GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS, 1); // 无 RPC 也发
builder.AddChannelArgument(GRPC_ARG_HTTP2_MAX_PINGS_WITHOUT_DATA, 0);   // 允许无限 PING
builder.AddChannelArgument(GRPC_ARG_HTTP2_MIN_RECV_PING_INTERVAL_WITHOUT_DATA_MS,
    std::chrono::duration_cast<std::chrono::milliseconds>(std::chrono::minutes(5)).count());

// 限制单连接最大并发流数
builder.AddChannelArgument(GRPC_ARG_MAX_CONCURRENT_STREAMS, 100);
```

### 1.2 客户端配置

```cpp
grpc::ChannelArguments args;
args.SetInt(GRPC_ARG_KEEPALIVE_TIME_MS, 30000);
args.SetInt(GRPC_ARG_KEEPALIVE_TIMEOUT_MS, 10000);
args.SetInt(GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS, 1);
auto channel = grpc::CreateCustomChannel("server:50051",
    grpc::InsecureChannelCredentials(), args);
```

---

## 2. 健康检查（Health Checking）

gRPC 标准健康检查协议，用于 LB/服务发现判断节点是否可用。

```protobuf
// grpc/health/v1/health.proto
service Health {
    rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
    rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}
```

```cpp
#include <grpcpp/health_check_service_interface.h>

// Server 端：注册健康检查服务
grpc::EnableDefaultHealthCheckService(true);

// 手动控制状态
auto* health = grpc::health::HealthCheckServiceInterface::Get();
health->SetServingStatus("", grpc::health::v1::HealthCheckResponse::SERVING);
health->SetServingStatus("myapp.MyService",
    grpc::health::v1::HealthCheckResponse::NOT_SERVING);

// K8s 中配合 gRPC 健康探针:
// livenessProbe/readinessProbe + grpc_health_probe 工具
```

```yaml
# K8s Pod spec
livenessProbe:
  exec:
    command: ["/grpc_health_probe", "-addr=:50051"]
  initialDelaySeconds: 10
  periodSeconds: 30
```

---

## 3. Channel 与 Stub 管理

### 3.1 Channel 是连接池

```
Channel = 对同一个目标服务器的 HTTP/2 连接池（可包含多个 TCP 连接）

准则:
- 一个目标服务 → 一个 Channel（复用，不要每次请求创建）
- Channel 是线程安全的，多线程共享
- Stub 是轻量的，可以每种 Stub 创建一个
```

```cpp
// ✅ 正确：在初始化时创建，全局共享
class MyClient {
    std::shared_ptr<grpc::Channel> channel_;
    std::unique_ptr<MyService::Stub> stub_;
public:
    MyClient(const std::string& addr)
        : channel_(grpc::CreateChannel(addr, grpc::InsecureChannelCredentials()))
        , stub_(MyService::NewStub(channel_)) {}

    Status DoSomething(const Request& req, Response* resp) {
        grpc::ClientContext ctx;
        ctx.set_deadline(std::chrono::system_clock::now() + std::chrono::seconds(3));
        return stub_->DoSomething(&ctx, req, resp);
    }
};

// ❌ 错误：每次请求创建 Channel（开销巨大）
```

### 3.2 Channel 状态管理

```cpp
// 监控连接状态
auto state = channel_->GetState(false);  // false = 不阻塞
switch (state) {
    case GRPC_CHANNEL_IDLE:        // 空闲（未建立连接）
    case GRPC_CHANNEL_CONNECTING:  // 正在连接
    case GRPC_CHANNEL_READY:       // 就绪
    case GRPC_CHANNEL_TRANSIENT_FAILURE: // 临时失败（会重试）
    case GRPC_CHANNEL_SHUTDOWN:    // 已关闭
}

// 等待状态变化（如等待 READY）
channel_->WaitForConnected(deadline);
```

---

## 4. 生产环境配置 Checklist

```cpp
// Server
grpc::ServerBuilder builder;
builder.AddChannelArgument(GRPC_ARG_KEEPALIVE_TIME_MS, 30000);
builder.AddChannelArgument(GRPC_ARG_KEEPALIVE_TIMEOUT_MS, 10000);
builder.AddChannelArgument(GRPC_ARG_MAX_CONCURRENT_STREAMS, 1000);   // 限制并发流
builder.AddChannelArgument(GRPC_ARG_MAX_RECEIVE_MESSAGE_LENGTH, 8*1024*1024); // 8MB
builder.AddChannelArgument(GRPC_ARG_MAX_SEND_MESSAGE_LENGTH, 8*1024*1024);

// Client
grpc::ChannelArguments args;
args.SetInt(GRPC_ARG_KEEPALIVE_TIME_MS, 30000);
args.SetInt(GRPC_ARG_KEEPALIVE_TIMEOUT_MS, 10000);
args.SetInt(GRPC_ARG_MAX_RECEIVE_MESSAGE_LENGTH, 8*1024*1024);
args.SetLoadBalancingPolicyName("round_robin");
```

---

## 5. gRPC 反射（Server Reflection）

启用反射后，`grpcurl` 等工具可以动态发现服务方法，无需 `.proto` 文件：

```cpp
#include <grpcpp/ext/proto_server_reflection_plugin.h>

grpc::reflection::InitProtoReflectionServerBuilderPlugin();
// 现在可以用 grpcurl 调试了
```

```bash
# 列出所有服务
grpcurl -plaintext localhost:50051 list

# 调用方法
grpcurl -plaintext -d '{"a":3,"b":4}' localhost:50051 Calculator/Add
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| Keepalive | PING 探测，防止连接被代理静默关闭 |
| 健康检查 | gRPC 标准 Health 协议，配合 K8s 探针 |
| Channel 复用 | 一个目标服务一个 Channel，线程安全 |
| 最大消息大小 | 默认 4MB，生产环境需设置 |
| 反射 | 启用后可用 grpcurl 调试 |

---

## 第十三章小结

| # | 篇目 | 核心 |
|---|------|------|
| 01 | 四种调用模式与拦截器 | Unary/ServerStream/ClientStream/Bidi + 拦截器链 |
| 02 | Deadline 与负载均衡 | Deadline传播/客户端LB/round_robin/重试策略 |
| 03 | 生产实践 | Keepalive/健康检查/Channel复用/反射 |
