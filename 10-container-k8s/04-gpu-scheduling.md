# 04 — GPU 调度：MIG、Device Plugin、GPU 共享

> 关联篇目：[02 核心资源](./02-k8s-core-resources.md)

---

## 1. Device Plugin——让 K8s 认识 GPU

K8s 原生只认识 CPU 和内存。GPU 通过 **Device Plugin** 注册为可调度资源。

```
┌──────────────────────────────┐
│         Kubelet               │
│  ┌────────────────────────┐  │
│  │  Device Plugin Manager   │  │
│  └────────┬───────────────┘  │
│           │ gRPC              │
│  ┌────────▼───────────────┐  │
│  │  NVIDIA Device Plugin   │  │ ← 向 kubelet 注册 GPU 资源
│  │  "nvidia.com/gpu"      │  │
│  └────────────────────────┘  │
└──────────────────────────────┘

Pod 申请 GPU:
resources:
  limits:
    nvidia.com/gpu: 2    # 2 张 GPU
```

---

## 2. 安装组件

```bash
# NVIDIA GPU Operator（一键部署所有 GPU 组件）
helm install gpu-operator nvidia/gpu-operator

# 包含:
# - NVIDIA Driver (内核模块)
# - NVIDIA Container Toolkit (nvidia-container-runtime)
# - NVIDIA Device Plugin (K8s 资源注册)
# - NVIDIA DCGM (监控)
```

### 2.1 时间片共享（Time-Slicing）

多个 Pod **时分复用**同一张 GPU：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
data:
  any: |-
    version: v1
    sharing:
      timeSlicing:
        resources:
        - name: nvidia.com/gpu
          replicas: 4    # 一张 GPU 虚拟成 4 份
```

```bash
# 然后 Pod 可以申请
resources:
  limits:
    nvidia.com/gpu: 1    # 实际用 1/4 GPU 时间片
```

| 优点 | 缺点 |
|------|------|
| 提高 GPU 利用率 | 无显存隔离，无故障隔离 |
| 适合小模型推理 | 一个 Pod OOM 影响同卡其他 Pod |

---

## 3. MIG——真正的硬件隔离

**M**ulti-**I**nstance **G**PU。从硬件层面将一张 GPU 切分成多个独立实例。

```
NVIDIA A100 (80GB):

MIG 配置:
┌──────────────────────────────┐
│ Instance 1: 10GB  (1g.5gb)   │ ← 独立显存 + 独立缓存 + 独立 SM
│ Instance 2: 20GB  (2g.10gb)  │
│ Instance 3: 20GB  (2g.10gb)  │
│ Instance 4: 30GB  (3g.20gb)  │
└──────────────────────────────┘

每个 Instance 互相完全隔离（OOM 不影响其他）
```

```bash
# 启用 MIG
nvidia-smi -mig 1

# 创建 MIG 实例
nvidia-smi mig -cgi 9,9,9,9  # 4 个等大实例
```

```yaml
# Pod 申请 MIG 设备
resources:
  limits:
    nvidia.com/mig-2g.10gb: 1
```

### 3.1 MIG vs Time-Slicing vs vGPU

| | Time-Slicing | MIG | vGPU (GRID) |
|------|:---:|:---:|:---:|
| 隔离级别 | 无 | **硬件级** | 驱动级 |
| 显存隔离 | ❌ | ✅ | ✅ |
| 故障隔离 | ❌ | ✅ | 部分 |
| GPU 型号 | 大部分 | A100/A30/H100 | 数据中心 GPU |
| License | 免费 | 免费 | 收费 |

---

## 4. AI 推理的 GPU 调度最佳实践

```
场景 1: 多模型小推理（如 10 个小模型）
→ Time-Slicing 或 MIG（硬件隔离更好）

场景 2: 大模型独占（如 Llama-70B）
→ 整卡 GPU 或 NVLink 多卡绑定

场景 3: 在线 + 离线混部
→ MIG 隔离在线推理和离线训练/批处理

场景 4: 弹性推理
→ KEDA + GPU HPA（根据请求队列长度自动扩缩 GPU Pod）
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| Device Plugin | 向 K8s 注册 GPU 为可调度资源 |
| Time-Slicing | 时分复用，无隔离，提高利用率 |
| MIG | 硬件级 GPU 切片，独立显存+SM |
| GPU Operator | NVIDIA 一键部署全栈 |

下一篇 [05 Helm 与 Kustomize](./05-helm-kustomize.md)。
