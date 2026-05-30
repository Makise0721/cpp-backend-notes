# 01 — Docker 原理：namespace、cgroup、UnionFS、镜像分层

> 关联篇目：[第四章 10 cgroups/namespace](../04-operating-systems/10-system-basics.md) | [02 K8s 核心资源](./02-k8s-core-resources.md)

---

## 1. 容器 vs 虚拟机

```
虚拟机:                         容器:
┌───────────┐                  ┌───────────┐
│ App       │                  │ App       │
├───────────┤                  ├───────────┤
│ Guest OS  │ ← 完整 OS 内核    │ 容器引擎   │
├───────────┤                  ├───────────┤
│ Hypervisor│                  │ 共享内核   │ ← 共用 Host Kernel
├───────────┤                  ├───────────┤
│ Host OS   │                  │ Host OS   │
└───────────┘                  └───────────┘

启动: 分钟级                    启动: 秒级
大小: GB 级                    大小: MB 级
隔离: 完全                      隔离: 进程级（共享内核）
```

---

## 2. 三大基石：namespace + cgroup + UnionFS

### 2.1 Namespace——隔离"你能看到什么"

| Namespace | 隔离内容 |
|-----------|----------|
| **PID** | 进程树隔离（容器内 PID 1 看不到宿主机进程） |
| **NET** | 独立网络栈（网卡、IP、端口、路由表、iptables） |
| **MNT** | 独立的文件系统挂载点（容器有自己的 `/`） |
| **UTS** | 主机名和域名 |
| **IPC** | System V IPC、POSIX 消息队列 |
| **USER** | UID/GID 映射（容器内 root ↔ 宿主机普通用户） |
| **Cgroup** | cgroup 视图隔离 |

```bash
# 查看某进程的 namespace
ls -l /proc/<PID>/ns/
```

### 2.2 Cgroups——限制"你能用多少资源"

```
cgroup 控制组:

/sys/fs/cgroup/cpu/docker/<container_id>/
├── cpu.shares      # CPU 权重（相对）
├── cpu.cfs_quota_us # CPU 配额（绝对，如 200000 = 2 核）
├── cpu.cfs_period_us

/sys/fs/cgroup/memory/docker/<container_id>/
├── memory.limit_in_bytes  # 内存上限
├── memory.usage_in_bytes  # 当前使用量
```

```bash
# Docker 资源限制本质上是写 cgroup 文件
docker run --cpus=2 --memory=4g my_image
```

### 2.3 UnionFS——镜像分层的秘密

**镜像由多层只读层 + 一层可写层叠在一起。**

```
容器文件系统:
┌────────────────────┐
│  容器层 (可写)       │ ← 运行时修改都写在这里（写时复制）
├────────────────────┤
│  Layer 3: 应用代码   │ ← RUN cp app /app
├────────────────────┤
│  Layer 2: 依赖库    │ ← RUN pip install -r requirements.txt
├────────────────────┤
│  Layer 1: Python基础 │ ← FROM python:3.11
├────────────────────┤
│  Layer 0: Ubuntu基础 │ ← 最底层
└────────────────────┘

每一层对应 Dockerfile 中的一条指令
每层只读，只有最上层可写
```

**写时复制（CoW）**：修改文件时，先将文件从只读层复制到可写层，在可写层中修改。不同容器共享相同的只读层。

---

## 3. Dockerfile 最佳实践

```dockerfile
# 多阶段构建（减小最终镜像）
# Stage 1: 构建
FROM gcc:13 AS builder
COPY . /src
WORKDIR /src
RUN make

# Stage 2: 运行（只复制必需产物）
FROM ubuntu:22.04
COPY --from=builder /src/my_app /app/my_app
CMD ["/app/my_app"]

# 层优化
# ✅ 先 COPY 依赖文件（不常变的层在前面，利用缓存）
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build
```

```bash
# 分析镜像层
docker history my_image
dive my_image  # 第三方工具，可视化分析每层大小
```

---

## 4. 容器运行时与 OCI

| 组件 | 作用 |
|------|------|
| **runc** | OCI 标准容器运行时（使用 libcontainer 创建容器） |
| **containerd** | 管理容器生命周期（在 runc 之上） |
| **Docker Engine** | 在 containerd 之上 + 构建/镜像管理 |
| **CRI-O** | Kubernetes 的轻量容器运行时 |

```
Docker 架构:
Docker CLI → dockerd → containerd → containerd-shim → runc → 容器进程
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| namespace | 进程级隔离（PID/NET/MNT/...） |
| cgroup | 资源限制和统计 |
| UnionFS | 分层文件系统 + CoW |
| 镜像分层 | Dockerfile 每条指令 = 一层 |
| 多阶段构建 | 分离构建环境和运行环境 |

下一篇 [02 K8s 核心资源](./02-k8s-core-resources.md)。
