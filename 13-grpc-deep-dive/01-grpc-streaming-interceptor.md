# 01 — gRPC 四种调用模式与拦截器

> 关联篇目：[第六章 09 Protobuf](../06-network-programming/09-high-performance.md) | [02 Deadline](./02-grpc-deadline-lb.md)

---

## 1. gRPC 概览

```
gRPC = Protobuf (序列化) + HTTP/2 (传输) + 多语言代码生成

Client                            Server
  │                                 │
  │ Stub (自动生成)                   │ Service (自动生成)
  │ add(a,b) ──── Protobuf ────→    │ Add(a,b)
  │             HTTP/2 帧            │
  │ ←──────── result ────────────── │
```

---

## 2. 四种调用模式

### 2.1 Unary（一元调用）——一问一答

```protobuf
service Calculator {
    rpc Add(AddRequest) returns (AddResponse);
}
```

```cpp
// Client
AddResponse resp;
grpc::ClientContext ctx;
grpc::Status status = stub_->Add(&ctx, request, &response);

// Server
grpc::Status Add(grpc::ServerContext* ctx,
                 const AddRequest* req, AddResponse* resp) override {
    resp->set_result(req->a() + req->b());
    return grpc::Status::OK;
}
```

### 2.2 Server Streaming——一问多答

```protobuf
rpc ListUsers(ListRequest) returns (stream User);
```

```cpp
// Client: 用 Reader 接收流
auto reader = stub_->ListUsers(&ctx, request);
User user;
while (reader->Read(&user)) {  // 逐个读取
    process(user);
}
grpc::Status status = reader->Finish();

// Server: 用 Writer 发送流
grpc::Status ListUsers(grpc::ServerContext* ctx,
                        const ListRequest* req,
                        grpc::ServerWriter<User>* writer) override {
    for (const auto& user : db->GetAll()) {
        writer->Write(user);
    }
    return grpc::Status::OK;
}
```

**场景**：批量导出数据、实时推送（日志/监控）。

### 2.3 Client Streaming——多问一答

```protobuf
rpc Upload(stream Chunk) returns (UploadStatus);
```

```cpp
// Client: 逐个发送
auto writer = stub_->Upload(&ctx, &status);
for (auto& chunk : chunks) {
    writer->Write(chunk);
}
writer->WritesDone();

// Server: 逐个接收
grpc::Status Upload(grpc::ServerContext* ctx,
                     grpc::ServerReader<Chunk>* reader,
                     UploadStatus* status) override {
    Chunk chunk;
    while (reader->Read(&chunk)) {
        assemble(chunk);
    }
    // 全部接收后才返回
    return grpc::Status::OK;
}
```

**场景**：文件上传、批量写入。

### 2.4 Bidi Streaming——双向流（最灵活）

```protobuf
rpc Chat(stream Message) returns (stream Message);
```

```cpp
// Client
auto stream = stub_->Chat(&ctx);
std::thread writer([&] {
    for (auto& msg : outgoing) stream->Write(msg);
    stream->WritesDone();
});
Message msg;
while (stream->Read(&msg)) { process(msg); }
writer.join();

// Server
grpc::Status Chat(grpc::ServerContext* ctx,
                   grpc::ServerReaderWriter<Message, Message>* stream) override {
    Message msg;
    // 可以用两个线程分别读和写
    while (stream->Read(&msg)) {
        stream->Write(generateReply(msg));
    }
    return grpc::Status::OK;
}
```

**场景**：聊天、实时协作、设备双向控制。

---

## 3. 拦截器（Interceptor）——gRPC 的中间件

### 3.1 一元拦截器

```cpp
// Server 端拦截器
class LoggingInterceptor : public grpc::ServerInterceptor {
public:
    void Intercept(grpc::experimental::InterceptorBatchMethods* methods) override {
        // 调用前
        auto start = std::chrono::steady_clock::now();

        methods->Proceed();  // 执行实际调用

        // 调用后
        auto elapsed = std::chrono::steady_clock::now() - start;
        std::cout << "RPC took " << elapsed.count() << "ns\n";
    }
};

// 注册
grpc::ServerBuilder builder;
builder.experimental().SetInterceptorCreators(
    std::make_unique<LoggingInterceptorFactory>());
```

### 3.2 拦截器链

```
请求 → [Auth拦截器] → [Log拦截器] → [RateLimit拦截器] → 实际处理
         ↓                ↓               ↓
       验证token       记录耗时         检查限流
```

### 3.3 常见拦截器场景

| 拦截器 | 作用 |
|--------|------|
| **认证** | 验证 JWT/OAuth token |
| **日志** | 记录 RPC 请求/响应和耗时 |
| **限流** | 令牌桶 / 滑动窗口 |
| **重试** | 透明重试失败的 RPC |
| **链路追踪** | 注入 trace_id 到 metadata |
| **错误转换** | 统一错误码映射 |

---

## 小结

| 模式 | 特点 | 适合 |
|------|------|------|
| Unary | 一问一答 | 大多数 API |
| Server Stream | 服务端持续发送 | 批量导出、推送 |
| Client Stream | 客户端持续发送 | 上传、批量写入 |
| Bidi Stream | 双向独立读写 | 聊天、实时协作 |
| Interceptor | 中间件链 | Auth/Log/Limit/Trace |

下一篇 [02 Deadline、超时控制与负载均衡](./02-grpc-deadline-lb.md)。
