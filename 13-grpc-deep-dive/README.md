# 第十三章：gRPC 深入

> 3 篇 · 从四种调用模式到生产级最佳实践。

---

## 篇目

| # | 文件 | 核心内容 |
|---|------|----------|
| 01 | [01-grpc-streaming-interceptor.md](./01-grpc-streaming-interceptor.md) | Unary/ServerStream/ClientStream/Bidi、拦截器链(Auth/Log/Limit) |
| 02 | [02-grpc-deadline-lb.md](./02-grpc-deadline-lb.md) | Deadline 传播(防止级联超时)、gRPC 客户端 LB(round_robin)、重试策略 |
| 03 | [03-grpc-production.md](./03-grpc-production.md) | Keepalive(PING 探测)、健康检查(Health Check)、Channel 复用、反射调试 |
