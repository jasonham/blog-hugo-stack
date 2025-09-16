---
title: 容器化
description: Docker镜像优化、K8s部署（Deployment/Service/Ingress）、HPA自动扩缩容
date: 2024-12-14T01:46:00+08:00
lastmod: 2024-12-14T01:46:00+08:00
slug: how-to-containerization

tags:
  - Docker
  - K8s
  - HPA
categories:
  - Docker
---

## Docker 镜像优化（Image Optimization）

###为什么需要优化？

- **镜像体积大** → 传输慢、启动慢、占用存储多
- **构建时间长** → 影响 CI/CD 效率
- **安全风险高** → 包含不必要的软件包或漏洞

---

### 优化手段详解：

#### 选择合适的基础镜像

- 避免使用 `ubuntu:latest` 或 `python:3.9`（完整版），改用：
  - `python:3.9-slim`（官方精简版）
  - `alpine`（极小，但注意兼容性，如 glibc 问题）
  - `distroless`（Google 出品，无 shell，极安全，适合生产）

> 示例：

```dockerfile
FROM python:3.9-slim  ## 而非 python:3.9
```

#### 合并 RUN 指令，减少镜像层

- Docker 镜像是分层的，每一层都会增加体积和构建时间
- 合并多个命令，用 `&&` 连接，并在最后清理缓存

> 示例：

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    pip install -r requirements.txt && \
    apt-get remove -y gcc && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*
```

#### 使用 .dockerignore

- 类似 `.gitignore`，避免把本地开发文件（如 `.git`, `__pycache__`, `venv`）打包进镜像

> 示例 `.dockerignore`：

```
.git
__pycache__
*.pyc
.env
venv/
tests/
```

#### 利用构建缓存（Build Cache）

- Docker 会缓存每一层，如果某层没变，后续层可复用缓存
- 把**不常变动的指令放前面**（如安装系统依赖、pip install），**常变动的放后面**（如 COPY 源码）

> 示例：

```dockerfile
COPY requirements.txt .
RUN pip install -r requirements.txt  ## 这层缓存可长期复用

COPY . .  ## 源码经常变，放后面
```

#### 多阶段构建（Multi-stage Build）

- 用一个“构建阶段”编译或安装依赖，另一个“运行阶段”只拷贝最终产物，丢弃中间文件

> 示例（golang）：

```dockerfile
## 构建阶段
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o myapp .

## 运行阶段
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/myapp .
EXPOSE 8080
CMD ["./myapp"]
```

#### 扫描镜像安全漏洞

- 使用 `docker scan` 或 Trivy、Clair 等工具扫描基础镜像是否含 CVE 漏洞

---

## K8s 部署（Deployment / Service / Ingress）

Kubernetes（K8s）是容器编排的事实标准。 目标**将应用从单机容器部署到集群化、服务化、网络化环境**。

---

### Deployment —— 管理 Pod 副本与滚动更新

#### 作用：

- 定义应用的**期望状态**（如：运行 3 个副本）
- 支持**滚动更新、回滚、自愈**（Pod 挂了自动拉起）

#### 示例 YAML：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-python-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-python-app
  template:
    metadata:
      labels:
        app: my-python-app
    spec:
      containers:
        - name: app
          image: my-registry/my-python-app:v1.0
          ports:
            - containerPort: 8000
          resources:
            limits:
              memory: "512Mi"
              cpu: "500m"
```

#### 关键能力：

- 如何做**滚动更新**？（修改 image tag，kubectl apply）
- 如何**回滚**？`kubectl rollout undo deployment/my-python-app`
- 如何查看状态？`kubectl rollout status deployment/my-python-app`

---

### Service —— 服务发现与负载均衡

#### 作用：

- 为一组 Pod 提供**稳定的访问入口（ClusterIP）**
- 支持**负载均衡、服务发现**

#### 类型：

- `ClusterIP`（默认，集群内访问）
- `NodePort`（节点端口暴露，测试用）
- `LoadBalancer`（云厂商提供外网 IP）
- `ExternalName`（映射到外部 DNS）

#### 示例（ClusterIP）：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-python-service
spec:
  selector:
    app: my-python-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: ClusterIP
```

> 其他 Pod 可通过 `http://my-python-service` 访问你的应用！

---

### Ingress —— 七层路由与域名管理

#### 作用：

- 对外暴露 HTTP/HTTPS 服务
- 支持**基于域名/路径的路由、TLS 终止、负载均衡**

#### 依赖：

- 需要集群中部署 **Ingress Controller**（如 Nginx Ingress、Traefik、AWS ALB）

#### 示例：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: api.mycompany.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: my-python-service
                port:
                  number: 80
  tls:
    - hosts:
        - api.mycompany.com
      secretName: my-tls-secret
```

> 用户访问 `https://api.mycompany.com/v1/xxx` → 路由到你的 Python 服务！

---

## HPA 自动扩缩容（Horizontal Pod Autoscaler）

#### 为什么需要 HPA？

- 流量高峰自动扩容，节省资源；低峰自动缩容，降低成本
- 实现“弹性伸缩”，应对突发流量（如秒杀、节假日）

---

### 工作原理：

HPA 根据**监控指标**（如 CPU、内存、或自定义指标如 QPS）自动调整 Deployment 的 `replicas` 数量。

---

### 配置步骤：

#### 确保已部署 Metrics Server（用于采集 Pod 指标）

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

#### 创建 HPA 策略

##### 基于 CPU 利用率：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-python-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-python-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

> 含义：当 Pod 平均 CPU 使用率 > 70%，自动扩容，最多 10 个副本；低于则缩容，最少 2 个。

##### 基于自定义指标（如 QPS）—— 需 Prometheus + Keda 或 Custom Metrics API

> 示例（使用 KEDA）：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-python-app-scaler
spec:
  scaleTargetRef:
    name: my-python-app
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.default.svc:9090
        metricName: http_requests_total
        query: sum(rate(http_requests_total{job="my-python-app"}[2m]))
        threshold: "100" ## 每秒100请求扩容
```

---

### HPA 调优建议：

- 设置合理的 `min/max replicas`，避免震荡
- 指标采集间隔不宜太短（默认 15-30 秒）
- 结合 `readinessProbe` 避免未就绪 Pod 被计入负载
- 监控扩容事件：`kubectl describe hpa my-python-app-hpa`

---

## 总结

> **容器化不是“能跑起来就行”，而是要“跑得快、跑得省、跑得稳”——从镜像瘦身到 K8s 编排再到智能扩缩容，每一步都体现工程深度。**
