# 05 — Helm 与 Kustomize：K8s 配置管理

> 关联篇目：[02 核心资源](./02-k8s-core-resources.md)

---

## 1. Helm——K8s 的"包管理器"

### 1.1 核心概念

```
Helm Chart = 模板 + 默认值 + 元数据

mychart/
├── Chart.yaml           # 元信息（名称、版本、依赖）
├── values.yaml          # 默认配置值
├── templates/
│   ├── deployment.yaml  # 模板（Go template 语法）
│   ├── service.yaml
│   └── _helpers.tpl     # 辅助模板函数
└── charts/              # 子 Chart 依赖
```

### 1.2 模板语法

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        {{- if .Values.resources }}          # 条件渲染
        resources:
{{ toYaml .Values.resources | indent 10 }}   # 嵌套值 YAML 化
        {{- end }}
```

```yaml
# values.yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.25"
resources:
  limits:
    cpu: "1"
    memory: "512Mi"

# 安装时覆盖值
helm install myapp ./mychart --set image.tag=1.26 --set replicaCount=5
```

### 1.3 Release 管理

```bash
helm install myapp ./mychart         # 安装（创建一个 Release）
helm upgrade myapp ./mychart         # 升级
helm rollback myapp 1                # 回滚到第 1 版
helm uninstall myapp                 # 删除
helm list                            # 查看所有 Release
helm history myapp                   # 查看 Release 历史
```

---

## 2. Kustomize——声明式配置叠加

**不需要模板引擎，通过"Overlay 叠加 Base"的方式管理多环境配置。**

```
base/                       # 基础配置
├── kustomization.yaml
├── deployment.yaml
└── service.yaml

overlays/
├── dev/                    # 开发环境 overlay
│   ├── kustomization.yaml
│   └── replica-patch.yaml
└── prod/                   # 生产环境 overlay
    ├── kustomization.yaml
    └── resource-patch.yaml
```

```yaml
# base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml

# overlays/prod/kustomization.yaml
bases:
- ../../base
patches:
- path: resource-patch.yaml
images:
- name: nginx
  newTag: "1.26"
namePrefix: prod-
```

```bash
# 渲染完整 YAML（不部署）
kubectl kustomize overlays/prod/

# 直接部署
kubectl apply -k overlays/prod/
```

---

## 3. Helm vs Kustomize

| | Helm | Kustomize |
|------|:---:|:---:|
| 方式 | 模板引擎 | 声明式 patch + overlay |
| 学习曲线 | 中（Go template） | 低（纯 YAML） |
| 多环境 | `-f values-prod.yaml` | `overlays/prod/` |
| 复用 | Chart 仓库 | Git 仓库 |
| K8s 集成 | 外部工具 | `kubectl apply -k` 原生支持 |
| 适合 | 应用打包分发 + 版本管理 | GitOps 多环境管理 |

**现代实践**：两者可以共存——用 Helm 打包应用，用 Kustomize 管理环境差异。

---

## 小结

| 概念 | 一句话 |
|------|--------|
| Helm Chart | 模板 + values.yaml → 渲染 K8s YAML |
| Release | Helm 安装的实例，支持回滚 |
| Kustomize | Base + Overlay 叠加，无模板，纯 YAML |
| GitOps | Kustomize 天然适合（ArgoCD/Flux） |

---

## 第十二章小结

| # | 篇目 | 核心 |
|---|------|------|
| 01 | Docker 原理 | namespace/cgroup/UnionFS/镜像分层/多阶段构建 |
| 02 | K8s 核心资源 | Pod/Deployment/Service/ConfigMap/Ingress |
| 03 | K8s 网络与存储 | CNI/Service类型/PV+PVC/StorageClass |
| 04 | GPU 调度 | Device Plugin/Time-Slicing/MIG/GPU Operator |
| 05 | Helm & Kustomize | Chart 模板/Release/Overlay/GitOps |
