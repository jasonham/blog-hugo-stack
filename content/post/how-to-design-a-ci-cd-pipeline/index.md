---
title: 自动化集成交付部署
description: CI/CD流水线设计（Github Actions/Jenkins）、自动化测试、灰度发布、回滚机制
date: 2025-01-17T12:46:58+08:00
lastmod: 2025-01-17T12:46:58+08:00
slug: how-to-design-a-ci-cd-pipeline

tags:
  - Github Actions
  - Jenkins
  - 自动化测试
  - 灰度发布
  - 回滚机制
categories:
  - CI/CD
---

## CI/CD 流水线设计（Github Actions / Jenkins）

### 什么是 CI/CD？

- **CI（Continuous Integration，持续集成）**：开发人员频繁（如每天多次）将代码合并到主干分支，每次合并后自动触发构建和测试，尽早发现集成错误。
- **CD（Continuous Delivery / Deployment，持续交付/部署）**：
- **持续交付**：代码在通过所有测试后，随时可手动部署到生产环境。
- **持续部署**：代码在通过所有测试后，自动部署到生产环境（无需人工干预）。

### 流水线（Pipeline）是什么？

流水线是自动化执行的一系列步骤，通常包括：
{{<mermaid>}}
flowchart LR
A[代码拉取] --> B[依赖安装] --> C[编译构建] --> D[单元测试] --> E[集成测试] --> F[打包] --> G[部署] --> H[通知]
{{</mermaid>}}

### 工具对比：Github Actions vs Jenkins

| 特性     | Github Actions               | Jenkins                              |
| -------- | ---------------------------- | ------------------------------------ |
| 部署方式 | 云原生，与 GitHub 深度集成   | 自托管，需自行维护服务器             |
| 配置文件 | `.github/workflows/*.yml`    | `Jenkinsfile`（Groovy 语法）         |
| 易用性   | 上手简单，适合中小项目       | 功能强大，插件丰富，适合复杂企业场景 |
| 扩展性   | 依赖 GitHub 生态，扩展有限   | 插件生态庞大，高度可定制             |
| 成本     | 免费额度充足，私有仓库也免费 | 免费开源，但需自备服务器和维护成本   |
| 触发机制 | Push、PR、定时、手动等       | 同上，支持 Webhook、定时、远程触发等 |

**推荐选择**：

- 实际工作中，我们会选择混合使用：`Actions` 管理代码/DEV 环境 + `Jenkins` 负责生产环境管理复杂流水线。

### 示例：Github Actions 简单流水线

```yaml
# .github/workflows/ci-cd.yml
name: Deploy Django App

on:
  push:
    branches: [dev] # 推送到 dev 分支时触发

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.10"

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run tests (optional but recommended)
        run: python manage.py test

      - name: Deploy via SSH and restart Gunicorn
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /home/deploy/myproject
            git pull origin dev
            source venv/bin/activate
            pip install -r requirements.txt
            python manage.py migrate --noinput
            python manage.py collectstatic --noinput
            sudo systemctl restart gunicorn
            sudo systemctl status gunicorn --no-pager
```

---

## 自动化测试

### 为什么需要自动化测试？

- 保证每次代码变更不影响已有功能（回归测试）
- 快速反馈，缩短修复周期
- 为持续部署提供质量门禁（Quality Gate）
- 减少人工测试成本

### 常见测试类型（按阶段）

| 类型       | 描述                                | 执行阶段       |
| ---------- | ----------------------------------- | -------------- |
| 单元测试   | 测试单个函数/类，Mock 依赖          | CI 构建阶段    |
| 集成测试   | 测试模块间交互，如 API、数据库      | CI 构建阶段    |
| 端到端测试 | 模拟用户操作整个系统（如 Selenium） | 部署后测试环境 |
| 性能测试   | 压力、负载、并发测试                | 预发布阶段     |
| 安全测试   | 扫描漏洞（如 OWASP ZAP、Snyk）      | 构建或部署阶段 |

### 如何集成到 CI/CD？

- 在流水线中添加测试步骤
- 测试失败 → 流水线中断，通知开发者
- 测试覆盖率报告（如使用 `coverage`）

  **最佳实践**：

- 测试用例要覆盖核心路径和边界条件
- 使用测试框架（如 Pytest、GoConvey ）
- 并行执行测试加速流程
- 失败时提供清晰日志和截图（E2E 测试）

---

## 灰度发布（Gray Release / Canary Release）

### 什么是灰度发布？

灰度发布是一种**渐进式发布策略**，先让一小部分用户或服务器使用新版本，观察运行情况（如性能、错误率、用户体验），再逐步扩大范围，最终全量上线。

### 为什么需要灰度发布？

- 降低新版本上线风险
- 快速发现问题并回滚，影响范围小
- 支持 A/B 测试、功能开关、用户分群

### 实现方式

#### 方式一：基于流量比例（最常见）

- 使用网关/负载均衡（如 traefik、Istio、Spring Cloud Gateway）控制流量比例
- 例如：5% 用户 → v2.0，95% → v1.0

#### 方式二：基于用户特征

- 根据用户 ID、地域、设备、VIP 等路由到不同版本
- 适合定向测试或内测

#### 方式三：基于服务器分组

- 部署新版本到部分服务器，逐步替换旧服务器

### 示例： traefik 灰度配置

#### Traefik 初始配置

```yaml
# docker-compose.yml
version: "3.8"

services:
  traefik:
    image: traefik:v2.10
    container_name: traefik
    command:
      - --api.insecure=true # 启用非安全的 API 端口
      - --providers.docker
      - --providers.file.directory=/etc/traefik/dynamic_conf # 监听这个目录
      - --providers.file.watch=true # 监控文件变化
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./dynamic_conf:/etc/traefik/dynamic_conf # 将本地目录映射到容器内
```

#### 动态配置文件

```yaml
# ./dynamic_conf/config.yml
http:
  routers:
    app-router:
      rule: "Host(`yourdomain.com`)"
      service: app-service

  services:
    app-service:
      loadBalancer:
        sticky:
          cookie: {}
        servers:
          - url: "http://app-v1:80"
            weight: 90 # 初始权重
          - url: "http://app-v2:80"
            weight: 10 # 初始权重


  # 下面是 Traefik API 调用的目标服务
  # 这里的配置可以省略，因为我们可以直接通过 API 创建和更新
  # app-v1-service:
  #   loadBalancer:
  #     servers:
  #       - url: "http://app-v1:80"
  #
  # app-v2-service:
  #   loadBalancer:
  #     servers:
  #       - url: "http://app-v2:80"
```

#### 灰度发布脚本

```bash
#canary-api-deploy.sh
#!/bin/bash

set -e

# 确保脚本有执行权限
# chmod +x canary-api-deploy.sh

# 检查参数
if [ "$#" -ne 2 ]; then
    echo "使用方法: $0 <v1_weight> <v2_weight>"
    echo "示例: $0 80 20  # 设置v1权重为80, v2为20"
    exit 1
fi

V1_WEIGHT=$1
V2_WEIGHT=$2
TRAEFIK_API_URL="http://localhost:8080/api/providers/file" # Traefik API 地址

# 检查权重总和是否为100
if [ $((V1_WEIGHT + V2_WEIGHT)) -ne 100 ]; then
    echo "错误: v1和v2的权重总和必须为100。"
    exit 1
fi

# 构建新的动态配置 JSON
NEW_CONFIG_JSON=$(cat <<EOF
{
  "http": {
    "services": {
      "app-service": {
        "loadBalancer": {
          "sticky": {
            "cookie": {}
          },
          "servers": [
            {
              "url": "http://app-v1:80",
              "weight": $V1_WEIGHT
            },
            {
              "url": "http://app-v2:80",
              "weight": $V2_WEIGHT
            }
          ]
        }
      }
    }
  }
}
EOF
)

# 使用 curl 发送 PUT 请求更新配置
curl -X PUT -H "Content-Type: application/json" -d "$NEW_CONFIG_JSON" $TRAEFIK_API_URL

echo "已通过 Traefik API 动态更新权重为: v1:$V1_WEIGHT% 和 v2:$V2_WEIGHT%。"
echo "请检查 Traefik Dashboard (http://localhost:8080) 确认更新。"
```

#### 执行灰度发布

第一步 (90/10)：`./canary-api-deploy.sh 90 10`

第二步 (50/50)：`./canary-api-deploy.sh 50 50`

完成 (0/100)：`./canary-api-deploy.sh 0 100`

**最佳实践**：

- 配合监控系统（Prometheus + Grafana）观察指标
- 设置自动熔断：错误率 > 5% 自动回滚
- 灰度周期建议 1~3 天，逐步放量

---

## 回滚机制（Rollback Mechanism）

### 什么是回滚？

当新版本上线后出现严重问题（如崩溃、数据错误、性能下降），**快速恢复到上一个稳定版本**的操作。

### 为什么需要自动回滚？

- 人工回滚慢，影响用户体验和业务收入
- 自动化可减少 MTTR（平均修复时间）
- 与监控告警联动，实现“无人值守”发布

### 回滚策略

#### 基于版本的回滚

- 保留历史版本镜像/包（Docker Image、Jar、War）
- 部署系统支持一键切换版本（如 Helm、K8s Deployment）

#### 基于 Git 的回滚

- 回滚到上一个 Git Commit / Tag
- 重新触发 CI/CD 流水线部署旧版本

#### 自动回滚（推荐）

- 监控关键指标（错误率、延迟、CPU）
- 超过阈值 → 自动触发回滚脚本

### 示例：Kubernetes 回滚

```bash
# 查看发布历史
kubectl rollout history deployment/my-app

# 回滚到上一个版本
kubectl rollout undo deployment/my-app

# 回滚到指定版本
kubectl rollout undo deployment/my-app --to-revision=3
```

### 示例：自动回滚脚本（伪代码）

```python
if get_error_rate() > 0.05:  # 错误率 > 5%
    trigger_rollback()
    send_alert("Auto rollback triggered due to high error rate!")
```

**最佳实践**：

- 回滚操作必须幂等、快速、可审计
- 回滚后保留日志，便于事后分析
- 定期演练回滚流程（如 Chaos Engineering）
- 数据库变更需特别小心（一般不自动回滚 DDL）

---

## 总结：四者如何协同工作？

一个完整的现代 DevOps 发布流程如下：
{{<mermaid>}}
flowchart TD
A[开发者提交代码] --> B[触发 CI 流水线<br>Github Actions/Jenkins]
B --> C[运行自动化测试<br>单元/集成/E2E]
C --> D{测试是否通过?}
D -->|是| E[打包构建 → 推送镜像/包]
D -->|否| F[通知失败并停止]
E --> G[CD 阶段: 部署到预发布环境<br>→ 自动化验收测试]
G --> H[灰度发布到生产<br>5% → 20% → 50% → 100%]
H --> I[监控系统实时采集指标<br>错误率、延迟、资源]
I --> J{指标是否异常?}
J -->|是| k[自动回滚至上一版本]
J -->|否| L[发布成功]
L --> M[通知团队 + 生成报告]
{{</mermaid>}}

---

## 最佳实践建议

1. **基础设施即代码（IaC）**：用 Terraform、Ansible 管理环境
2. **不可变基础设施**：每次部署新建实例，避免配置漂移
3. **蓝绿部署 / 金丝雀发布**：结合灰度策略
4. **日志 + 监控 + 告警三位一体**：ELK + Prometheus + AlertManager
5. **流水线可视化**：让团队随时看到构建/部署状态
6. **权限与审批**：生产环境部署需人工审批（尤其金融、医疗行业）
