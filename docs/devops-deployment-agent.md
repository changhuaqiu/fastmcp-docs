# DevOps 部署 Agent 设计文档

> 基于 IaC 的代码化部署智能助手

---

## 目录

1. [架构概述](#架构概述)
2. [Agent 设计](#agent-设计)
3. [场景与用户故事](#场景与用户故事)
4. [Skill 设计](#skill-设计)
5. [MCP 能力拆分](#mcp-能力拆分)
6. [工作流设计](#工作流设计)
7. [文件结构](#文件结构)
8. [交互示例](#交互示例)

---

## 架构概述

```
┌─────────────────────────────────────────────────────────────┐
│                      用户交互层                              │
│                    Claude / 开发者                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      Agent 层                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                    deploy.md                           │ │
│  │              统一部署 Agent (智能路由)                   │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      Skill 层                              │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐  │
│  │  iac   │ │ build  │ │ deploy │ │ canary │ │ monitor │  │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘  │
│  ┌────────┐ ┌────────┐ ┌────────┐                         │
│  │  env   │ │approval│ │   ai   │                         │
│  └────────┘ └────────┘ └────────┘                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    MCP 能力层 (多源)                         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │ devops-mcp  │ │  k8s-mcp    │ │jenkins-mcp  │          │
│  │ (通用能力)   │ │ (K8s能力)   │ │ (构建能力)   │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │prometheus-  │ │  harbor-mcp │ │   git-mcp   │          │
│  │   mcp       │ │ (镜像仓库)   │ │ (代码仓库)   │          │
│  │ (监控能力)   │ │             │ │             │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    基础设施层                                │
│  K8s │ Jenkins │ Harbor │ GitLab │ Prometheus │ ArgoCD     │
└─────────────────────────────────────────────────────────────┘
```

---

## Agent 设计

### deploy.md - 统一部署 Agent

```markdown
# Deploy Agent - 统一部署智能助手

## 概述
基于 IaC 的代码化部署智能助手，帮助开发者完成从资源生成、校验、部署到灰度发布的全流程操作。

## 核心能力
- **IaC 管理**：解析、验证、渲染 IaC 模板
- **构建部署**：自动化构建、部署到 K8s
- **环境管理**：多环境配置与管理
- **灰度发布**：金丝雀部署、流量控制
- **智能评估**：AI 驱动的灰度效果评估
- **监控运维**：健康检查、日志分析、指标监控

## 支持的场景

| 场景 | 关键词 | 工作流 |
|------|--------|--------|
| 资源生成代码 | 生成、创建、IaC、配置 | code-gen |
| 代码模版校验 | 检查、验证、校验、lint | validate |
| 快速部署验证 | 部署、dev、测试、验证 | quick-deploy |
| 部署环境管理 | 环境、创建、配置、管理 | env-manage |
| 灰度部署 | 灰度、金丝雀、canary | canary |
| 灰度智能评估 | 评估、分析、效果、指标 | assess |
| 正式环境部署 | 生产、prod、上线、发布 | prod-deploy |

## 意图识别规则

### code-gen (资源生成代码)
- "为 xxx 服务生成 K8s 配置"
- "创建部署文件"
- "生成 IaC 代码"
- "渲染 Helm 模板"

### validate (代码模版校验)
- "检查配置"
- "验证 IaC"
- "校验模板"
- "lint 检查"
- "安全扫描"

### quick-deploy (快速部署验证)
- "部署到 dev"
- "快速验证"
- "测试部署"
- "发布到测试环境"

### env-manage (部署环境管理)
- "创建环境"
- "管理环境"
- "查看环境状态"
- "配置环境变量"

### canary (灰度部署)
- "灰度发布"
- "金丝雀部署"
- "发布 20% 流量"
- "canary deploy"

### assess (灰度智能评估)
- "评估灰度"
- "灰度效果"
- "分析灰度数据"
- "检查灰度状态"

### prod-deploy (正式环境部署)
- "发布生产"
- "生产部署"
- "上线"
- "prod release"
```

---

## 场景与用户故事

### 场景 1：资源生成代码

**用户故事**
> 作为一名后端开发工程师，我刚完成了一个新的微服务开发，需要生成对应的 K8s 部署配置文件。我希望能够基于服务的基本信息（如服务名、端口、资源需求），自动生成符合规范的 K8s YAML 文件，包括 Deployment、Service、ConfigMap 等，而不需要手动编写这些复杂的配置。

**典型对话**
```
开发者: "为 user-service 生成 K8s 部署配置，端口 8080，需要 2 个副本，内存 512Mi"

Agent:
[解析服务信息]
- 服务名: user-service
- 端口: 8080
- 副本数: 2
- 内存限制: 512Mi

[生成配置文件]
✓ deployment.yaml - Deployment 配置
✓ service.yaml - Service 配置
✓ configmap.yaml - ConfigMap 配置
✓ hpa.yaml - 自动扩缩容配置

[验证生成结果]
✓ 通过 K8s Schema 验证
✓ 通过最佳实践检查

配置文件已生成到 ./k8s/dev/user-service/

需要我帮你部署到 dev 环境验证吗？
```

**工作流**
```
用户输入 → 意图识别 → code-gen 工作流
    ↓
1. iac.parse      → 读取服务配置
2. iac.render     → 渲染 K8s 模板
3. iac.validate   → 验证生成代码
4. iac.lint       → 最佳实践检查
    ↓
输出: 生成的 YAML 文件 + 验证报告
```

---

### 场景 2：代码模版校验

**用户故事**
> 作为一名 DevOps 工程师，我需要确保团队提交的 IaC 配置文件符合公司规范和安全标准。我希望在配置合并到主分支前，能够自动检查配置的正确性、安全性以及是否符合最佳实践，及早发现潜在问题。

**典型对话**
```
开发者: "检查我的 IaC 配置有没有问题"

Agent:
[扫描 IaC 文件]
发现配置文件:
- k8s/prod/user-service.yaml
- terraform/prod/main.tf
- helm/user-service/values.yaml

[执行检查]
├─ Schema 验证      ✓ 通过
├─ 安全扫描        ⚠ 发现 2 个警告
│  └─ 容器以 root 运行
│  └─ 未设置资源限制
├─ 最佳实践检查    ⚠ 发现 3 个建议
│  └─ 缺少 livenessProbe
│  └─ 缺少 readinessProbe
│  └─ 建议添加 PodDisruptionBudget
└─ 策略合规检查    ✓ 通过

[生成修复建议]
我可以帮你自动修复这些问题，是否继续？
```

**工作流**
```
用户输入 → 意图识别 → validate 工作流
    ↓
1. iac.parse       → 扫描 IaC 文件
2. iac.validate    → Schema 验证
3. iac.lint        → 最佳实践检查
4. iac.security    → 安全扫描
5. iac.compliance  → 策略合规检查
    ↓
输出: 校验报告 + 问题列表 + 修复建议
```

---

### 场景 3：快速部署验证

**用户故事**
> 作为一名开发者，我完成代码开发后，希望能够快速部署到测试环境验证功能。我希望能够一键完成构建、推送镜像、更新部署的全流程，而不需要在多个系统之间切换操作。

**典型对话**
```
开发者: "部署 user-service 到 dev 环境验证"

Agent:
[发布前检查]
✓ IaC 配置验证通过
✓ 代码编译通过
✓ 单元测试通过

[开始部署流程]
├─ [1/4] 触发 Jenkins 构建...
│  ✓ 构建 #1234 已启动
│  ✓ 镜像: harbor.example.com/user-service:abc123
│  ✓ 构建完成 (耗时 2分30秒)
│
├─ [2/4] 推送镜像到仓库...
│  ✓ 镜像已推送
│
├─ [3/4] 更新 K8s 部署...
│  ✓ 更新 Deployment
│  ✓ 等待 Pod 就绪...
│  ✓ 2/2 Pods ready
│
└─ [4/4] 健康检查...
   ✓ /health 端点正常
   ✓ 服务响应正常

[部署完成]
✓ user-service 已部署到 dev 环境
✓ 访问地址: http://user-service.dev.example.com
✓ 日志地址: http://logs.dev.example.com/user-service

需要查看日志或进行接口测试吗？
```

**工作流**
```
用户输入 → 意图识别 → quick-deploy 工作流
    ↓
1. iac.validate      → 验证 IaC 配置
2. build.trigger     → 触发 Jenkins 构建
3. build.status      → 等待构建完成
4. image.push        → 推送镜像
5. deploy.k8s        → 部署到 K8s
6. health.check      → 健康检查
    ↓
输出: 部署状态 + 访问地址 + 日志链接
```

---

### 场景 4：部署环境管理

**用户故事**
> 作为一名平台工程师，我需要管理多个部署环境（dev、test、staging、prod）。我希望能够方便地创建新环境、配置环境变量、查看环境状态，以及比较不同环境之间的配置差异。

**典型对话**
```
开发者: "创建一个新的 staging 环境"

Agent:
[环境规划]
基于 dev 环境配置创建 staging 环境

[创建环境资源]
├─ [1/5] 创建 K8s namespace...
│  ✓ namespace/staging 已创建
│
├─ [2/5] 创建配置...
│  ✓ ConfigMap 已创建
│  ✓ Secret 已创建
│
├─ [3/5] 配置网络策略...
│  ✓ NetworkPolicy 已创建
│  ✓ Ingress 已创建
│
├─ [4/5] 配置资源配额...
│  ✓ ResourceQuota: CPU 10核, 内存 20Gi
│  ✓ LimitRange: 默认 CPU 500m, 内存 512Mi
│
└─ [5/5] 配置监控告警...
   ✓ ServiceMonitor 已创建
   ✓ PrometheusRule 已创建

[环境创建完成]
✓ staging 环境已创建
环境信息:
- 集群: k8s-staging.example.com
- 命名空间: staging
- Ingress: *.staging.example.com
- 监控: http://prometheus.staging.example.com

需要部署服务到 staging 环境吗？
```

**工作流**
```
用户输入 → 意图识别 → env-manage 工作流
    ↓
1. env.check         → 检查环境是否存在
2. env.create        → 创建命名空间和基础资源
3. env.configure     → 配置环境变量和 ConfigMap
4. env.network       → 配置网络策略和 Ingress
5. env.quota         → 配置资源配额
6. env.monitor       → 配置监控和告警
7. env.status        → 确认环境状态
    ↓
输出: 环境信息 + 配置详情
```

---

### 场景 5：灰度部署

**用户故事**
> 作为一名发布负责人，我需要在生产环境进行灰度发布，逐步放量验证新版本。我希望能够灵活控制灰度流量（如 5%、10%、20%），实时监控灰度版本的表现，并在出现问题时快速回滚。

**典型对话**
```
开发者: "对 user-service 进行灰度发布，先放 10% 流量"

Agent:
[发布前检查]
✓ 当前版本: v1.2.0
✓ 灰度版本: v1.3.0
✓ IaC 配置验证通过

[准备灰度部署]
├─ [1/5] 部署灰度版本...
│  ✓ canary Deployment 已创建
│  ✓ 1 个 canary Pod 已启动
│  ✓ canary Pod ready
│
├─ [2/5] 配置流量分割...
│  ✓ 90% → stable (v1.2.0)
│  ✓ 10% → canary (v1.3.0)
│
├─ [3/5] 配置监控...
│  ✓ canary 专属监控指标已创建
│  ✓ 错误率告警已配置
│
├─ [4/5] 记录基线指标...
│  ✓ 请求量: 1000 req/min
│  ✓ 错误率: 0.1%
│  ✓ P99 延迟: 120ms
│
└─ [5/5] 开始实时监控...
   监控中...

[灰度部署完成]
✓ canary 版本已部署，10% 流量
✓ 监控面板: http://grafana.example.com/canary/user-service

当前状态:
- stable: v1.2.0 (90% 流量)
- canary: v1.3.0 (10% 流量)

需要我持续监控并智能评估吗？或者你想手动调整流量？
```

**工作流**
```
用户输入 → 意图识别 → canary 工作流
    ↓
1. iac.validate        → 验证 IaC 配置
2. canary.prepare      → 准备灰度环境
3. deploy.canary       → 部署灰度版本
4. traffic.split       → 配置流量分割
5. monitor.baseline    → 记录基线指标
6. monitor.start       → 开始实时监控
    ↓
输出: 灰度状态 + 监控链接 + 流量配置
```

---

### 场景 6：灰度智能评估

**用户故事**
> 作为一名 SRE，我需要评估灰度发布的效果，判断是否可以继续扩大流量或需要回滚。我希望系统能够自动收集各项指标（错误率、延迟、日志等），通过 AI 分析给出专业的评估建议。

**典型对话**
```
开发者: "评估一下 user-service 的灰度效果"

Agent:
[收集灰度数据]
观察时间: 30分钟
数据源: Prometheus + 日志系统 + 链路追踪

[指标对比分析]
┌─────────────┬─────────────┬─────────────┬──────────┐
│     指标    │   stable    │   canary    │   差异   │
├─────────────┼─────────────┼─────────────┼──────────┤
│ 请求量      │  900/min    │  100/min    │  -       │
│ 错误率      │  0.10%      │  0.08%      │  ✓ -20%  │
│ P50 延迟    │  45ms       │  42ms       │  ✓ -7%   │
│ P99 延迟    │  120ms      │  115ms      │  ✓ -4%   │
│ CPU 使用率  │  65%        │  62%        │  ✓ -5%   │
│ 内存使用    │  450Mi      │  440Mi      │  ✓ -2%   │
└─────────────┴─────────────┴─────────────┴──────────┘

[日志分析]
✓ 无新增错误日志
✓ 无异常堆栈
✓ 无告警触发

[AI 评估]
基于收集的数据，AI 分析结果:

✓ 性能表现: canary 版本各项指标均优于或等于 stable
✓ 稳定性: 错误率下降 20%，无明显异常
✓ 资源使用: CPU 和内存使用略有下降

[评估结论]
🟢 建议继续推进

可以执行以下操作:
1. 扩大灰度流量到 50%
2. 直接全量发布
3. 继续观察当前状态

需要我帮你执行哪个操作？
```

**工作流**
```
用户输入 → 意图识别 → assess 工作流
    ↓
1. metrics.collect     → 收集 Prometheus 指标
2. logs.analyze        → 分析日志
3. traces.analyze      → 分析链路追踪
4. metrics.compare     → 对比 stable 和 canary
5. ai.evaluate         → AI 综合评估
    ↓
输出: 评估报告 + 建议 + 可执行操作
```

---

### 场景 7：正式环境部署

**用户故事**
> 作为一名发布经理，我需要将经过验证的版本发布到生产环境。这个过程需要严格的审批流程、全面的发布前检查、平滑的部署过程，以及完善的监控和快速回滚能力。

**典型对话**
```
开发者: "准备将 user-service v1.3.0 发布到生产环境"

Agent:
[发布前检查清单]
├─ [1/6] 代码审查
│  ✓ MR 已被批准
│  ✓ 2 名 reviewer 已通过
│
├─ [2/6] 测试验证
│  ✓ 单元测试通过
│  ✓ 集成测试通过
│  ✓ 压力测试通过
│  ✓ 安全扫描通过
│
├─ [3/6] 灰度验证
│  ✓ 灰度发布已完成
│  ✓ 灰度评估通过
│  ✓ 无回滚事件
│
├─ [4/6] 配置检查
│  ✓ 生产环境 IaC 配置正确
│  ✓ 环境变量已配置
│  ✓ 密钥已更新
│
├─ [5/6] 容量规划
│  ✓ 当前容量充足
│  ✓ HPA 已配置
│  ✓ 告警阈值已设置
│
└─ [6/6] 发布窗口
   ✓ 当前处于允许发布时段
   ✓ 无其他发布冲突

[创建审批流程]
需要以下人员审批:
├─ 技术负责人 (@tech-lead)
├─ 运维负责人 (@ops-lead)
└─ 安全负责人 (@security)

已发送审批通知，等待审批...

[30分钟后]
✓ 所有审批已完成

[开始生产部署]
├─ [1/4] 备份当前版本...
│  ✓ 配置已备份
│  ✓ 数据库 schema 已记录
│
├─ [2/4] 执行部署...
│  ✓ 更新 Deployment
│  ✓ 滚动更新中...
│  ✓ Pod 逐个替换中...
│  ✓ 所有 Pod 更新完成
│
├─ [3/4] 部署验证...
│  ✓ 健康检查通过
│  ✓ 冒烟测试通过
│  ✓ 关键接口验证通过
│
└─ [4/4] 监控设置...
   ✓ 5分钟监控窗口已启动
   ✓ 告警规则已激活
   ✓ 自动回滚已启用

[部署完成]
🎉 user-service v1.3.0 已成功发布到生产环境

部署信息:
- 版本: v1.3.0
- 时间: 2025-03-20 14:30:00
- 执行人: @developer
- 变更记录: https://git.example.com/commits/v1.3.0

当前状态:
- 所有 Pod: Ready
- 健康检查: Normal
- 错误率: 0.05% (低于告警阈值)

监控面板: http://grafana.example.com/production/user-service
```

**工作流**
```
用户输入 → 意图识别 → prod-deploy 工作流
    ↓
1. pre_deploy.check    → 发布前检查
2. approval.request    → 创建审批流程
3. approval.wait       → 等待审批完成
4. deploy.backup       → 备份当前版本
5. deploy.k8s          → 执行 K8s 部署
6. deploy.verify       → 部署验证
7. monitor.setup       → 设置监控和告警
8. monitor.observe     → 观察期监控
    ↓
输出: 部署结果 + 监控链接 + 变更记录
```

---

## Skill 设计

### IaC Skills

#### iac/parse.md
**职责**: 解析 IaC 文件（Terraform/Helm/Kustomize/YAML）

**MCP 工具**:
- `devops-mcp.iac.parse_file` - 解析单个文件
- `devops-mcp.iac.parse_dir` - 解析整个目录

**输入**:
```yaml
input:
  type: "file" | "dir"
  path: "./k8s/prod"
  format: "yaml" | "terraform" | "helm" | "kustomize"
```

**输出**:
```yaml
result:
  resources:
    - kind: "Deployment"
      name: "user-service"
    - kind: "Service"
      name: "user-service"
  config:
    replicas: 3
    image: "harbor.example.com/user-service:v1.2.0"
```

---

#### iac/validate.md
**职责**: 验证 IaC 配置的正确性

**MCP 工具**:
- `devops-mcp.iac.validate_schema` - Schema 验证
- `devops-mcp.iac.validate_syntax` - 语法验证

---

#### iac/render.md
**职责**: 渲染模板生成最终配置

**MCP 工具**:
- `devops-mcp.iac.render_helm` - 渲染 Helm 模板
- `devops-mcp.iac.render_kustomize` - 渲染 Kustomize

---

#### iac/lint.md
**职责**: 最佳实践检查

**MCP 工具**:
- `devops-mcp.iac.lint` - Lint 检查
- `devops-mcp.iac.check_best_practices` - 最佳实践检查
- `devops-mcp.iac.security_scan` - 安全扫描

---

### Build Skills

#### build/build.md
**职责**: 触发和管理构建流程

**MCP 工具**:
- `jenkins-mcp.build.trigger` - 触发构建
- `jenkins-mcp.build.status` - 查询构建状态
- `jenkins-mcp.build.logs` - 获取构建日志

---

### Deploy Skills

#### deploy/k8s-deploy.md
**职责**: 部署到 Kubernetes

**MCP 工具**:
- `k8s-mcp.deploy.apply` - 应用配置
- `k8s-mcp.deploy.rollout_status` - 查询部署状态
- `k8s-mcp.deploy.rollback` - 回滚部署

---

#### deploy/rollback.md
**职责**: 部署回滚

**MCP 工具**:
- `k8s-mcp.deploy.rollback` - 回滚到指定版本
- `k8s-mcp.deploy.rollback_history` - 查询回滚历史

---

### Canary Skills

#### canary/prepare.md
**职责**: 准备灰度环境

**MCP 工具**:
- `k8s-mcp.canary.create` - 创建 canary deployment
- `k8s-mcp.canary.config` - 配置 canary 参数

---

#### canary/traffic.md
**职责**: 流量分割管理

**MCP 工具**:
- `k8s-mcp.traffic.split` - 配置流量分割
- `k8s-mcp.traffic.status` - 查询流量状态

---

#### canary/promote.md
**职责**: 灰度转正

**MCP 工具**:
- `k8s-mcp.canary.promote` - canary 转为 stable
- `k8s-mcp.canary.cleanup` - 清理 canary 资源

---

### Monitor Skills

#### monitor/health.md
**职责**: 健康检查

**MCP 工具**:
- `k8s-mcp.health.pod` - Pod 健康检查
- `k8s-mcp.health.service` - Service 健康检查
- `devops-mcp.health.endpoint` - 端点健康检查

---

#### monitor/metrics.md
**职责**: 指标收集与分析

**MCP 工具**:
- `prometheus-mcp.query` - 查询指标
- `prometheus-mcp.query_range` - 范围查询
- `prometheus-mcp.compare` - 指标对比

---

#### monitor/logs.md
**职责**: 日志获取与分析

**MCP 工具**:
- `devops-mcp.logs.fetch` - 获取日志
- `devops-mcp.logs.analyze` - 分析日志
- `devops-mcp.logs.search` - 搜索日志

---

### Env Skills

#### env/list.md
**职责**: 列出所有环境

**MCP 工具**:
- `devops-mcp.env.list` - 列出环境
- `k8s-mcp.namespace.list` - 列出命名空间

---

#### env/create.md
**职责**: 创建新环境

**MCP 工具**:
- `k8s-mcp.namespace.create` - 创建命名空间
- `k8s-mcp.quota.create` - 创建资源配额
- `k8s-mcp.configmap.create` - 创建配置

---

#### env/status.md
**职责**: 查询环境状态

**MCP 工具**:
- `k8s-mcp.namespace.get` - 获取命名空间信息
- `k8s-mcp.resource.list` - 列出资源

---

### Approval Skills

#### approval/flow.md
**职责**: 审批流程管理

**MCP 工具**:
- `devops-mcp.approval.create` - 创建审批
- `devops-mcp.approval.status` - 查询审批状态
- `devops-mcp.approval.notify` - 发送通知

---

### AI Skills

#### ai/evaluate.md
**职责**: AI 智能评估

**MCP 工具**:
- `ai-mcp.evaluate.canary` - 评估灰度效果
- `ai-mcp.evaluate.metrics` - 评估指标数据
- `ai-mcp.analyze.logs` - 分析日志
- `ai-mcp.suggest.action` - 建议下一步操作

---

## MCP 能力拆分

### MCP Server 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP Aggregator                           │
│                  (统一调用层)                                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      MCP Servers                            │
├─────────────┬─────────────┬─────────────┬─────────────────┤
│ devops-mcp  │  k8s-mcp    │ jenkins-mcp │ prometheus-mcp  │
│             │             │             │                 │
│ 通用能力     │ K8s 专用     │ 构建专用     │ 监控专用        │
├─────────────┼─────────────┼─────────────┼─────────────────┤
│ harbor-mcp  │   git-mcp   │ argo-mcp    │    ai-mcp       │
│             │             │             │                 │
│ 镜像仓库     │ 代码仓库     │ GitOps      │  AI 分析        │
└─────────────┴─────────────┴─────────────┴─────────────────┘
```

### devops-mcp (通用能力)

**职责**: 提供 DevOps 通用能力，不依赖特定系统

**工具列表**:
```python
# IaC 相关
iac.parse_file(path, format)           # 解析 IaC 文件
iac.validate_schema(config)            # Schema 验证
iac.render_template(template, vars)    # 渲染模板
iac.lint(config)                       # Lint 检查
iac.security_scan(config)              # 安全扫描
iac.diff(env1, env2)                   # 配置差异

# 环境管理
env.list()                             # 列出环境
env.get(env_name)                      # 获取环境信息
.env.validate(config)                  # 验证环境配置

# 审批流程
approval.create(workflow, approvers)   # 创建审批
approval.status(approval_id)           # 查询状态
approval.notify(approval_id, message)  # 发送通知

# 日志
logs.fetch(service, env, lines)        # 获取日志
logs.search(query, time_range)         # 搜索日志
logs.analyze(logs)                     # 分析日志

# 健康检查
health.check(endpoint, timeout)        # 端点健康检查
health.check_service(service, env)     # 服务健康检查
```

---

### k8s-mcp (Kubernetes 能力)

**职责**: 提供 Kubernetes 专用能力

**工具列表**:
```python
# 部署相关
deploy.apply(manifest)                 # 应用配置
.deploy.rollout_status(deployment)     # 查询部署状态
deploy.rollback(deployment, revision)  # 回滚
deploy.scale(deployment, replicas)     # 调整副本数

# Pod 相关
pod.list(namespace, labels)            # 列出 Pod
pod.get(name, namespace)               # 获取 Pod 详情
pod.logs(name, namespace, tail)        # 获取 Pod 日志
pod.exec(name, namespace, command)     # 执行命令

# 命名空间相关
namespace.list()                       # 列出命名空间
namespace.get(name)                    # 获取命名空间
namespace.create(name, config)         # 创建命名空间
namespace.delete(name)                 # 删除命名空间

# 资源相关
resource.list(kind, namespace)         # 列出资源
resource.get(kind, name, namespace)    # 获取资源
resource.patch(kind, name, patch)      # 更新资源

# Config/Secret
configmap.create(name, data, ns)       # 创建 ConfigMap
configmap.update(name, data, ns)       # 更新 ConfigMap
secret.create(name, data, ns)          # 创建 Secret

# 灰度发布
canary.create(name, canary_config)     # 创建 canary
canary.delete(name)                    # 删除 canary
canary.status(name)                    # 查询 canary 状态
canary.promote(name)                   # canary 转正

# 流量管理
traffic.split(service, stable, canary, ratio)  # 配置流量分割
traffic.status(service)                # 查询流量状态

# 监控
health.pod(name, namespace)            # Pod 健康检查
health.service(name, namespace)        # Service 健康检查
```

---

### jenkins-mcp (Jenkins 能力)

**职责**: 提供 Jenkins 构建相关能力

**工具列表**:
```python
# 构建相关
build.trigger(job, parameters)          # 触发构建
build.status(job, build_id)            # 查询构建状态
build.stop(job, build_id)              # 停止构建
build.logs(job, build_id)              # 获取构建日志

# Job 相关
job.list()                             # 列出 Job
job.get(job_name)                      # 获取 Job 详情
job.build(job_name)                    # 构建 Job

# 参数相关
params.list(job_name)                  # 列出参数
params.validate(job_name, params)      # 验证参数
```

---

### prometheus-mcp (监控能力)

**职责**: 提供 Prometheus 监控相关能力

**工具列表**:
```python
# 查询相关
query.query(promql)                    # 即时查询
query.range(promql, start, end)        # 范围查询
query.series(match)                    # 查询时间序列

# 指标相关
metrics.list(match)                    # 列出指标
metrics.get(metric_name)               # 获取指标详情
metrics.query_metadata(metric)         # 查询元数据

# 对比分析
compare(metric, labels1, labels2)      # 对比指标
compare.baseline(metric, period)       # 与基线对比

# 告警相关
alert.list()                           # 列出告警
alert.get(rule_name)                   # 获取告警规则
alert.silence.create(matchers, duration)  # 创建静默
```

---

### harbor-mcp (镜像仓库能力)

**职责**: 提供 Harbor 镜像仓库相关能力

**工具列表**:
```python
# 镜像相关
image.list(project)                    # 列出镜像
image.get(image_name)                  # 获取镜像详情
image.tags(image_name)                 # 获取镜像标签
image.delete(image_name, tag)          # 删除镜像

# 推送相关
registry.push(image, tag)              # 推送镜像
registry.pull(image, tag)              # 拉取镜像
registry.manifest(image, tag)          # 获取镜像 Manifest

# 项目相关
project.list()                         # 列出项目
project.create(name)                   # 创建项目
project.delete(name)                   # 删除项目
```

---

### git-mcp (代码仓库能力)

**职责**: 提供 Git 仓库相关能力

**工具列表**:
```python
# 仓库相关
repo.list()                            # 列出仓库
repo.get(repo_name)                   # 获取仓库信息

# 分支相关
branch.list(repo)                      # 列出分支
branch.create(repo, name)              # 创建分支
branch.delete(repo, name)              # 删除分支

# Commit 相关
commit.list(repo, branch)              # 列出 Commit
commit.get(repo, sha)                  # 获取 Commit 详情
.commit.diff(repo, sha1, sha2)         # 对比差异

# MR/PR 相关
mr.list(repo)                          # 列出 MR
mr.get(repo, mr_id)                    # 获取 MR 详情
mr.create(repo, source, target)        # 创建 MR
mr.approve(repo, mr_id)                # 批准 MR
mr.merge(repo, mr_id)                  # 合并 MR
```

---

### argo-mcp (GitOps 能力)

**职责**: 提供 ArgoCD GitOps 相关能力

**工具列表**:
```python
# 应用相关
app.list()                             # 列出应用
app.get(app_name)                      # 获取应用详情
app.sync(app_name)                     # 同步应用
app.refresh(app_name)                  # 刷新应用

# 部署相关
deployment.history(app_name)           # 部署历史
deployment.rollback(app_name, revision) # 回滚

# 操作相关
operation.list(app_name)               # 列出操作
operation.wait(app_name, op_id)        # 等待操作完成
```

---

### ai-mcp (AI 分析能力)

**职责**: 提供 AI 智能分析相关能力

**工具列表**:
```python
# 评估相关
evaluate.canary(stable_metrics, canary_metrics)  # 评估灰度效果
evaluate.metrics(metrics)              # 评估指标数据
evaluate.logs(logs)                    # 分析日志
evaluate.traces(traces)                # 分析链路

# 分析相关
analyze.anomaly_detection(data)        # 异常检测
analyze.trend_analysis(metrics)        # 趋势分析
analyze.capacity_planning(data)        # 容量规划

# 建议相关
suggest.action(context)                # 建议操作
suggest.optimization(config)           # 优化建议
suggest.scaling(metrics)               # 扩容建议

# 预测相关
predict.traffic(history)               # 流量预测
.predict.failure(metrics)              # 故障预测
predict.capacity(history)              # 容量预测
```

---

## 工作流设计

### 1. code-gen 工作流

```yaml
name: code-gen
description: 资源生成代码工作流

steps:
  - name: 解析服务信息
    skill: iac/parse
    inputs:
      service_name: from_user
      port: from_user
      replicas: from_user
      resources: from_user

  - name: 渲染 K8s 模板
    skill: iac/render
    inputs:
      template: "k8s/deployment"
      vars: from_previous_step

  - name: 渲染 Service 模板
    skill: iac/render
    inputs:
      template: "k8s/service"
      vars: from_previous_step

  - name: 渲染 ConfigMap 模板
    skill: iac/render
    inputs:
      template: "k8s/configmap"
      vars: from_previous_step

  - name: 验证生成的配置
    skill: iac/validate
    inputs:
      config: from_previous_step

  - name: 最佳实践检查
    skill: iac/lint
    inputs:
      config: from_previous_step

  - name: 输出结果
    action: write_files
    outputs:
      - deployment.yaml
      - service.yaml
      - configmap.yaml
```

---

### 2. validate 工作流

```yaml
name: validate
description: 代码模版校验工作流

steps:
  - name: 扫描 IaC 文件
    skill: iac/parse
    inputs:
      path: "./iac"

  - name: Schema 验证
    skill: iac/validate
    inputs:
      config: from_previous_step

  - name: 语法检查
    skill: iac/lint
    inputs:
      config: from_previous_step
      check_type: "syntax"

  - name: 安全扫描
    skill: iac/lint
    inputs:
      config: from_previous_step
      check_type: "security"

  - name: 最佳实践检查
    skill: iac/lint
    inputs:
      config: from_previous_step
      check_type: "best_practices"

  - name: 策略合规检查
    skill: iac/validate
    inputs:
      config: from_previous_step
      policy: "company_policy"

  - name: 生成报告
    action: generate_report
    outputs:
      - validation_report.md
```

---

### 3. quick-deploy 工作流

```yaml
name: quick-deploy
description: 快速部署验证工作流

steps:
  - name: 验证 IaC 配置
    skill: iac/validate
    inputs:
      env: "dev"

  - name: 触发构建
    skill: build/build
    inputs:
      service: from_user
      branch: "main"
      env: "dev"

  - name: 等待构建完成
    skill: build/status
    inputs:
      build_id: from_previous_step
    wait_for: "success"

  - name: 推送镜像
    skill: build/push
    inputs:
      image: from_previous_step

  - name: 部署到 K8s
    skill: deploy/k8s-deploy
    inputs:
      service: from_user
      image: from_previous_step
      env: "dev"

  - name: 等待 Pod 就绪
    skill: monitor/health
    inputs:
      service: from_user
      env: "dev"
      check_type: "pod_ready"
    wait_for: "healthy"

  - name: 健康检查
    skill: monitor/health
    inputs:
      service: from_user
      env: "dev"
      check_type: "endpoint"

  - name: 输出部署信息
    action: summarize
    outputs:
      - status
      - endpoint
      - logs_url
```

---

### 4. env-manage 工作流

```yaml
name: env-manage
description: 部署环境管理工作流

steps:
  - name: 检查环境是否存在
    skill: env/status
    inputs:
      env: from_user

  - name: 创建命名空间
    skill: env/create
    inputs:
      env: from_user
    condition: "env_not_exists"

  - name: 创建 ConfigMap
    skill: env/create
    inputs:
      resource: "configmap"
      env: from_user

  - name: 创建 Secret
    skill: env/create
    inputs:
      resource: "secret"
      env: from_user

  - name: 配置网络策略
    skill: env/create
    inputs:
      resource: "networkpolicy"
      env: from_user

  - name: 配置 Ingress
    skill: env/create
    inputs:
      resource: "ingress"
      env: from_user

  - name: 配置资源配额
    skill: env/create
    inputs:
      resource: "resourcequota"
      env: from_user

  - name: 配置监控
    skill: env/create
    inputs:
      resource: "servicemonitor"
      env: from_user

  - name: 确认环境状态
    skill: env/status
    inputs:
      env: from_user

  - name: 输出环境信息
    action: summarize
    outputs:
      - env_info
```

---

### 5. canary 工作流

```yaml
name: canary
description: 灰度部署工作流

steps:
  - name: 验证 IaC 配置
    skill: iac/validate
    inputs:
      env: "prod"

  - name: 获取当前稳定版本
    skill: monitor/metrics
    inputs:
      service: from_user
      env: "prod"
      metric: "deployment_version"

  - name: 准备灰度环境
    skill: canary/prepare
    inputs:
      service: from_user
      stable_version: from_previous_step
      canary_version: from_user

  - name: 部署灰度版本
    skill: deploy/k8s-deploy
    inputs:
      deployment_type: "canary"
      service: from_user
      version: from_user

  - name: 配置流量分割
    skill: canary/traffic
    inputs:
      service: from_user
      stable_ratio: from_user
      canary_ratio: from_user

  - name: 记录基线指标
    skill: monitor/metrics
    inputs:
      service: from_user
      metrics:
        - request_rate
        - error_rate
        - latency_p50
        - latency_p99
      duration: "5m"

  - name: 开始实时监控
    skill: monitor/metrics
    inputs:
      service: from_user
      mode: "continuous"

  - name: 输出灰度状态
    action: summarize
    outputs:
      - canary_status
      - traffic_config
      - monitor_url
```

---

### 6. assess 工作流

```yaml
name: assess
description: 灰度智能评估工作流

steps:
  - name: 收集 stable 版本指标
    skill: monitor/metrics
    inputs:
      service: from_user
      version: "stable"
      metrics:
        - request_rate
        - error_rate
        - latency_p50
        - latency_p99
        - cpu_usage
        - memory_usage
      duration: "30m"

  - name: 收集 canary 版本指标
    skill: monitor/metrics
    inputs:
      service: from_user
      version: "canary"
      metrics:
        - request_rate
        - error_rate
        - latency_p50
        - latency_p99
        - cpu_usage
        - memory_usage
      duration: "30m"

  - name: 分析日志
    skill: monitor/logs
    inputs:
      service: from_user
      versions: ["stable", "canary"]
      duration: "30m"

  - name: 对比指标
    skill: monitor/metrics
    inputs:
      stable_metrics: from_step_1
      canary_metrics: from_step_2
      compare:
        - error_rate
        - latency
        - resource_usage

  - name: AI 评估
    skill: ai/evaluate
    inputs:
      metrics_comparison: from_previous_step
      log_analysis: from_step_3
      evaluation_type: "canary"

  - name: 生成建议
    skill: ai/evaluate
    inputs:
      evaluation_result: from_previous_step
      suggest_actions: true

  - name: 输出评估报告
    action: generate_report
    outputs:
      - assessment_report
      - recommendation
```

---

### 7. prod-deploy 工作流

```yaml
name: prod-deploy
description: 正式环境部署工作流

steps:
  - name: 发布前检查
    skill: iac/validate
    inputs:
      env: "prod"
      checks:
        - code_review
        - test_validation
        - security_scan
        - canary_validation

  - name: 创建审批流程
    skill: approval/flow
    inputs:
      workflow: "prod_deploy"
      approvers:
        - "tech_lead"
        - "ops_lead"
        - "security_lead"

  - name: 等待审批
    skill: approval/status
    inputs:
      approval_id: from_previous_step
    wait_for: "approved"

  - name: 备份当前版本
    skill: deploy/backup
    inputs:
      service: from_user
      env: "prod"

  - name: 执行部署
    skill: deploy/k8s-deploy
    inputs:
      service: from_user
      env: "prod"
      strategy: "rolling_update"

  - name: 部署验证
    skill: monitor/health
    inputs:
      service: from_user
      env: "prod"
      checks:
        - pod_ready
        - endpoint_health
        - smoke_test

  - name: 设置监控
    skill: monitor/metrics
    inputs:
      service: from_user
      env: "prod"
      alert_rules: true
      observation_window: "5m"

  - name: 观察期监控
    skill: monitor/health
    inputs:
      service: from_user
      env: "prod"
      mode: "observe"
      duration: "5m"

  - name: 输出部署报告
    action: generate_report
    outputs:
      - deployment_summary
      - monitor_urls
      - change_log
```

---

## 文件结构

```
devops-deployment/
├── README.md
├── deploy.md                          # 统一部署 Agent
│
├── skills/                            # Skill 定义
│   ├── iac/
│   │   ├── parse.md
│   │   ├── validate.md
│   │   ├── render.md
│   │   └── lint.md
│   ├── build/
│   │   ├── build.md
│   │   ├── status.md
│   │   └── push.md
│   ├── deploy/
│   │   ├── k8s-deploy.md
│   │   ├── rollback.md
│   │   └── backup.md
│   ├── canary/
│   │   ├── prepare.md
│   │   ├── traffic.md
│   │   └── promote.md
│   ├── monitor/
│   │   ├── health.md
│   │   ├── metrics.md
│   │   └── logs.md
│   ├── env/
│   │   ├── list.md
│   │   ├── create.md
│   │   └── status.md
│   ├── approval/
│   │   └── flow.md
│   └── ai/
│       ├── evaluate.md
│       └── suggest.md
│
├── mcp/                               # MCP Servers
│   ├── devops-mcp/
│   │   ├── server.py
│   │   └── tools/
│   ├── k8s-mcp/
│   ├── jenkins-mcp/
│   ├── prometheus-mcp/
│   ├── harbor-mcp/
│   ├── git-mcp/
│   ├── argo-mcp/
│   └── ai-mcp/
│
├── workflows/                         # 工作流定义
│   ├── code-gen.yaml
│   ├── validate.yaml
│   ├── quick-deploy.yaml
│   ├── env-manage.yaml
│   ├── canary.yaml
│   ├── assess.yaml
│   └── prod-deploy.yaml
│
├── templates/                         # IaC 模板
│   ├── k8s/
│   │   ├── deployment/
│   │   ├── service/
│   │   ├── configmap/
│   │   └── ingress/
│   ├── helm/
│   │   └── charts/
│   └── terraform/
│
├── config/                            # 配置文件
│   ├── environments.yaml
│   ├── policies.yaml
│   └── approvers.yaml
│
└── docs/                              # 文档
    ├── architecture.md
    ├── user-guide.md
    └── api-reference.md
```

---

## 交互示例

### 示例 1：完整的部署流程

```
开发者: "我想部署 user-service 到 dev 环境"

Agent:
好的，我来帮你部署 user-service 到 dev 环境。

[发布前检查]
✓ IaC 配置验证通过
✓ 当前 Git 分支: feature/new-api
✓ 最新 commit: abc1234 - Add new API endpoints

开始部署流程...

[1/5] 触发 Jenkins 构建
构建任务已创建: https://jenkins.example.com/job/user-service-dev/1234
等待构建完成...

✓ 构建成功
镜像: harbor.example.com/user-service:abc1234

[2/5] 推送镜像
✓ 镜像已推送到仓库

[3/5] 更新 K8s 部署
应用配置到 dev 环境...
✓ Deployment 已更新
✓ 等待 Pod 就绪...
✓ 2/2 Pods Ready

[4/5] 健康检查
检查 /health 端点...
✓ 服务健康

[5/5] 验证完成
✓ 部署成功

部署信息:
- 服务: user-service
- 环境: dev
- 版本: abc1234
- 副本数: 2/2
- 访问地址: http://user-service.dev.example.com
- 监控面板: http://grafana.dev.example.com/d/user-service

需要我帮你做什么？
1. 查看日志
2. 进行接口测试
3. 扩容副本数
```

---

### 示例 2：灰度发布和评估

```
开发者: "对 user-service 进行灰度发布，先 10% 流量"

Agent:
好的，我来帮你进行灰度发布。

[灰度前检查]
✓ 当前版本: v1.2.0 (stable)
✓ 新版本: v1.3.0 (canary)
✓ IaC 配置验证通过
✓ 生产环境资源充足

[准备灰度部署]
创建 canary deployment...
✓ canary Deployment 已创建
✓ 1 个 canary Pod 已启动
✓ canary Pod Ready

[配置流量分割]
当前流量配置:
- stable (v1.2.0): 90%
- canary (v1.3.0): 10%

✓ 流量规则已应用

[记录基线指标]
收集 stable 版本基线指标（5分钟）...
✓ 请求量: 1000 req/min
✓ 错误率: 0.1%
✓ P50 延迟: 45ms
✓ P99 延迟: 120ms

[开始监控]
已启动 canary 专属监控
监控面板: http://grafana.example.com/canary/user-service

灰度部署完成！我会持续监控 canary 版本的表现。

---
(30分钟后)

---

Agent:
灰度运行 30 分钟，我来评估一下效果。

[收集指标]
stable 版本 (90% 流量):
- 请求量: 900/min
- 错误率: 0.10%
- P50 延迟: 45ms
- P99 延迟: 120ms

canary 版本 (10% 流量):
- 请求量: 100/min
- 错误率: 0.08% ✓ 降低 20%
- P50 延迟: 42ms ✓ 降低 7%
- P99 延迟: 115ms ✓ 降低 4%

[日志分析]
✓ 无新增错误
✓ 无异常堆栈
✓ 无告警触发

[AI 评估]
基于收集的数据，评估结果:

🟢 性能表现: 优秀
- canary 版本各项指标均优于 stable

🟢 稳定性: 良好
- 错误率下降 20%

🟢 资源使用: 正常
- CPU/内存使用率略有下降

[评估结论]
🟢 建议继续推进

可执行操作:
1. 扩大灰度流量到 50%
2. 直接全量发布
3. 继续观察当前状态

你想执行哪个操作？
```

---

### 示例 3：生产部署

```
开发者: "准备将 user-service v1.3.0 发布到生产环境"

Agent:
好的，开始生产部署流程。

[发布前检查清单]
├─ [1/6] 代码审查
   ✓ MR 已被批准
   ✓ 2 名 reviewer 已通过
   ✓ 无代码冲突

├─ [2/6] 测试验证
   ✓ 单元测试通过 (98% 覆盖率)
   ✓ 集成测试通过
   ✓ 压力测试通过 (1000 QPS)
   ✓ 安全扫描通过

├─ [3/6] 灰度验证
   ✓ 灰度发布已完成
   ✓ 灰度评估通过
   ✓ 观察期无异常

├─ [4/6] 配置检查
   ✓ 生产环境 IaC 配置正确
   ✓ 环境变量已配置
   ✓ 密钥已更新

├─ [5/6] 容量规划
   ✓ 当前容量充足
   ✓ HPA 已配置 (2-10 副本)
   ✓ 告警阈值已设置

└─ [6/6] 发布窗口
   ✓ 当前处于允许发布时段
   ✓ 无其他发布冲突

所有检查通过！

[创建审批流程]
需要以下人员审批:
- @tech_lead (技术负责人)
- @ops_lead (运维负责人)
- @security_lead (安全负责人)

已发送审批通知...

---
(等待审批中)
---

Agent:
✓ @tech_lead 已批准
✓ @ops_lead 已批准
✓ @security_lead 已批准

所有审批已完成！开始部署...

[1/5] 备份当前版本
✓ 配置已备份到: /backups/user-service-20250320-143000
✓ 数据库 schema 已记录

[2/5] 执行滚动更新
更新 Deployment...
✓ 滚动更新策略: 25% max unavailable, 25% max surge
✓ 逐个替换 Pod 中...
   Pod 1/4: ✓ 更新完成
   Pod 2/4: ✓ 更新完成
   Pod 3/4: ✓ 更新完成
   Pod 4/4: ✓ 更新完成

[3/5] 部署验证
✓ 所有 Pod Ready
✓ /health 端点正常
✓ 冒烟测试通过

[4/5] 设置监控
✓ 5分钟监控窗口已启动
✓ 告警规则已激活:
   - 错误率 > 0.5%
   - P99 延迟 > 500ms
   - Pod 重启 > 0
✓ 自动回滚已启用

[5/5] 进入观察期
正在监控部署状态...

---
(5 分钟观察期后)
---

Agent:
🎉 部署成功！

部署摘要:
- 服务: user-service
- 环境: production
- 版本: v1.2.0 → v1.3.0
- 时间: 2025-03-20 14:30:00
- 执行人: @developer

当前状态:
- 所有 Pod: Ready (4/4)
- 健康检查: Normal
- 错误率: 0.05% ✓
- P99 延迟: 115ms ✓

观察期数据:
- 请求量: 1200 req/min
- 错误率: 0.05% (低于基线 0.10%)
- P50 延迟: 42ms
- P99 延迟: 115ms
- 无 Pod 重启
- 无告警触发

相关链接:
- 变更记录: https://git.example.com/commits/v1.3.0
- 监控面板: http://grafana.example.com/production/user-service
- 日志查询: http://logs.example.com/user-service
- 告警规则: http://prometheus.example.com/rules/user-service

部署完成！需要我帮你做什么？
```

---

## 总结

本文档定义了基于 IaC 的代码化部署智能助手的完整设计：

1. **统一 Agent (deploy.md)**: 通过意图识别路由到不同工作流
2. **7 大场景**: 覆盖从资源生成到生产部署的完整流程
3. **Skill 模块化**: 每个 Skill 负责单一功能，可复用
4. **MCP 多源集成**: 支持 8 个 MCP Server，覆盖不同系统能力
5. **工作流编排**: 每个场景有明确的工作流定义
6. **用户故事驱动**: 每个场景都有详细的用户故事和交互示例

这个架构设计具有良好的：
- **扩展性**: 新增场景只需添加工作流
- **复用性**: Skill 可在不同场景中复用
- **可维护性**: MCP Server 独立开发和部署
- **用户体验**: 一个入口，自动识别意图
