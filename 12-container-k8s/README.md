# 第十二章：容器与 Kubernetes

> 5 篇 · 从 Docker 原理到 GPU 调度，覆盖容器化与 K8s 编排的核心知识。

---

## 篇目

| # | 文件 | 核心内容 |
|---|------|----------|
| 01 | [01-docker-principles.md](./01-docker-principles.md) | namespace/cgroup/UnionFS、镜像分层、Dockerfile 多阶段构建、OCI 运行时 |
| 02 | [02-k8s-core-resources.md](./02-k8s-core-resources.md) | Pod、Deployment(滚动更新)、Service(ClusterIP/NodePort/LB)、ConfigMap、Ingress |
| 03 | [03-k8s-networking-storage.md](./03-k8s-networking-storage.md) | CNI(Calico/Flannel/Cilium)、PV/PVC、StorageClass(动态供给) |
| 04 | [04-gpu-scheduling.md](./04-gpu-scheduling.md) | NVIDIA Device Plugin、Time-Slicing、MIG(硬件级GPU切片) |
| 05 | [05-helm-kustomize.md](./05-helm-kustomize.md) | Helm Chart 模板语法、Release 管理、Kustomize Base+Overlay、GitOps |
