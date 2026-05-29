# 03 — K8s 网络与存储：CNI、Service 类型、PV/PVC

> 关联篇目：[02 核心资源](./02-k8s-core-resources.md)

---

## 1. K8s 网络模型——三个"必须"

1. Pod 之间互相通信（跨 Node 也可以）
2. Node 可以访问所有 Pod
3. Pod 看到自己的 IP 和其他 Pod 看到的 IP 相同

### 1.1 CNI——容器网络接口

K8s 不提供网络实现，交给 CNI 插件：

| CNI | 特点 | 适用 |
|-----|------|------|
| **Flannel** | 简单 Overlay（VXLAN） | 入门 |
| **Calico** | BGP 路由 + NetworkPolicy | **生产推荐**（高性能 + 网络策略） |
| **Cilium** | eBPF 驱动 | 高性能、可观测性 |
| **Weave** | Mesh 网络 | 中小规模 |

### 1.2 Service 类型

```yaml
# ClusterIP（默认）：集群内
type: ClusterIP

# NodePort：在每个 Node 上开一个端口（30000-32767）
type: NodePort
# 外部通过 <NodeIP>:<NodePort> 访问

# LoadBalancer：云厂商创建外部 LB
type: LoadBalancer
# 外部通过 LB 的 IP/域名访问

# ExternalName：DNS 别名（映射到外部服务）
type: ExternalName
externalName: external-db.example.com
```

```
外部流量 → LoadBalancer → [NodePort → ClusterIP → Pod]
           (云 LB)          (kube-proxy 转发)
```

---

## 2. 存储：PV（持久卷）与 PVC（持久卷声明）

**管理员创建 PV（实际的存储），用户创建 PVC（申请存储），K8s 自动绑定。**

```yaml
# PV：由管理员预先创建
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce          # RWO: 单节点读写
    # - ReadOnlyMany         # ROX: 多节点只读
    # - ReadWriteMany        # RWX: 多节点读写
  persistentVolumeReclaimPolicy: Retain  # Delete / Retain / Recycle
  nfs:
    path: /data
    server: 10.0.0.5

---
# PVC：用户申请存储
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi       # K8s 会自动找一个 ≥5G 的 PV 绑定

---
# Pod 使用 PVC
spec:
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

### 2.1 StorageClass——动态供给

不用手动创建 PV，K8s 根据 PVC 的 StorageClass **自动创建 PV**。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs   # 或 cephfs, nfs-client 等
parameters:
  type: gp3

---
# PVC 引用 StorageClass → 自动创建 PV
spec:
  storageClassName: fast-ssd
```

---

## 3. StatefulSet——有状态应用

与 Deployment 不同：Pod 有**固定标识**（`<name>-0`, `<name>-1`），稳定的网络 ID + 独立存储。

```
适用: 数据库、消息队列、ZooKeeper 等有状态服务
```

---

## 小结

| 概念 | 一句话 |
|------|--------|
| CNI | 容器网络接口（Calico/Flannel/Cilium） |
| ClusterIP/NodePort/LoadBalancer | 三种 Service 访问方式 |
| PV/PVC | 持久卷的声明与绑定 |
| StorageClass | 动态创建 PV |

下一篇 [04 GPU 调度](./04-gpu-scheduling.md)。
