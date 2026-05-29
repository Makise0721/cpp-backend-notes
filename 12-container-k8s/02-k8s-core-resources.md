# 02 — K8s 核心资源：Pod、Deployment、Service、ConfigMap、Ingress

> 关联篇目：[01 Docker 原理](./01-docker-principles.md) | [03 网络与存储](./03-k8s-networking-storage.md)

---

## 1. K8s 架构概览

```
┌─────────────────────────────────────────────┐
│              Control Plane                   │
│  ┌──────────┐ ┌──────────┐ ┌─────────────┐ │
│  │ API Server│ │Scheduler │ │Controller Mgr│ │
│  └──────────┘ └──────────┘ └─────────────┘ │
│  ┌──────────┐                               │
│  │   etcd   │ ← 集群状态存储                  │
│  └──────────┘                               │
└─────────────────────────────────────────────┘
          │
    ┌─────┴──────────────┐
    ▼                    ▼
┌─────────┐         ┌─────────┐
│ Node 1  │         │ Node 2  │
│ ┌─────┐ │         │ ┌─────┐ │
│ │kubelet│         │ │kubelet│
│ ├─────┤ │         │ ├─────┤ │
│ │ Pod │ │         │ │ Pod │ │
│ └─────┘ │         │ └─────┘ │
└─────────┘         └─────────┘
```

---

## 2. Pod——最小调度单元

```
Pod = 一个或多个共享网络/存储的容器

┌────────────── Pod ──────────────┐
│  ┌──────────┐    ┌──────────┐   │
│  │ 主容器    │    │ Sidecar  │   │
│  │ (业务)    │    │ (日志/代理)│   │
│  └──────────┘    └──────────┘   │
│      共享: localhost, Volume, IPC│
└─────────────────────────────────┘
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: myapp:v1
    ports:
    - containerPort: 8080
    resources:
      requests:  { cpu: "500m", memory: "256Mi" }  # 调度保证
      limits:    { cpu: "1",    memory: "512Mi" }  # 硬上限
```

---

## 3. Deployment——无状态应用的控制器

**声明期望的副本数，K8s 保证实际状态 = 期望状态。**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3                        # 3 个副本
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

**Rolling Update（滚动更新）**：逐步替换旧 Pod，零停机。

```bash
kubectl set image deployment/nginx-deploy nginx=nginx:1.26
kubectl rollout status deployment/nginx-deploy
kubectl rollout undo deployment/nginx-deploy  # 回滚
```

---

## 4. Service——稳定的访问入口

**Pod 的 IP 会变（重启/调度），Service 提供固定的 ClusterIP + DNS 名。**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
  - port: 80        # Service 端口
    targetPort: 8080 # Pod 端口
  type: ClusterIP   # 默认：仅集群内可访问
  # type: NodePort  # 也开放到每个 Node 的静态端口
  # type: LoadBalancer # 云厂商 LB
```

```bash
# 集群内访问
curl http://nginx-svc.default.svc.cluster.local
#        <服务名>.<命名空间>.svc.cluster.local
```

---

## 5. ConfigMap & Secret——配置与密钥分离

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "mysql://db:3306/mydb"
  log_level: "debug"
---
# 注入方式 1: 环境变量
env:
- name: DB_URL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: database_url

# 注入方式 2: 挂载为文件
volumeMounts:
- name: config
  mountPath: /etc/config
volumes:
- name: config
  configMap:
    name: app-config
```

```yaml
# Secret（Base64 编码存储，配合 RBAC + etcd 加密）
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: bXlwYXNzd29yZA==   # echo -n "mypassword" | base64
```

---

## 6. Ingress——HTTP 路由入口

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: api.example.com           # 域名路由
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-svc
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: order-svc
            port:
              number: 80
```

**Ingress Controller**（Nginx Ingress / Traefik / Istio Gateway）负责实际流量转发。

---

## 小结

| 资源 | 作用 | 核心 |
|------|------|------|
| Pod | 最小调度单元 | 共享网络/存储 |
| Deployment | 无状态副本管理 | 滚动更新/回滚 |
| Service | 稳定访问入口 | ClusterIP + DNS |
| ConfigMap | 配置分离 | 环境变量 / 挂载文件 |
| Ingress | HTTP 路由 | 域名+路径 → Service |

下一篇 [03 K8s 网络与存储](./03-k8s-networking-storage.md)。
