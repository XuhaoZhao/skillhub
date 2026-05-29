# SkillHub 模块架构文档

> 本文档系统梳理 SkillHub 项目的代码模块架构，帮助开发者快速理解整体设计。

---

## 一、项目总览

**SkillHub** 是一个企业级开源智能体技能注册中心，提供技能的发布、发现、审核和管理能力。

```
skillhub/
├── server/          # 后端（Java 21 / Spring Boot 3.2.3）
├── web/             # 前端（React 19 / TypeScript / Vite）
├── cli/             # 命令行工具（TypeScript / Bun）
├── scanner/         # 安全扫描器（Python / Cisco skill-scanner）
├── deploy/          # 部署配置（K8s / Kustomize）
├── monitoring/      # 监控（Prometheus + Grafana）
├── scripts/         # 运维脚本（smoke test / CI / 发布）
├── document/        # 用户文档站点（VitePress）
├── docs/            # 设计文档与 PRD
└── Makefile         # 40+ 统一构建命令
```

---

## 二、后端模块架构（server/）

采用 **DDD（领域驱动设计）+ 六边形架构**，多模块 Maven 项目。

### 2.1 模块依赖关系

```
                        ┌─────────────┐
                        │ skillhub-app│  ← 启动入口 / Web API / DTO / Controller
                        └──────┬──────┘
           ┌───────────┬───────────┼───────────┬──────────────┐
           ▼           ▼           ▼           ▼              ▼
   ┌──────────────┐ ┌────────┐ ┌────────┐ ┌──────────┐ ┌─────────────┐
   │skillhub-auth │ │skillhub│ │skillhub│ │skillhub  │ │skillhub-    │
   │  认证授权     │ │-search │ │-infra  │ │-notif    │ │  domain     │
   └──────┬───────┘ │  搜索   │ │ JPA实现│ │  通知     │ │  核心领域   │
          │         └──┬───┘ └───┬────┘ └─────┬────┘ └──────┬──────┘
          │            │         │            │             │
          └────────────┴─────────┴────────────┘             │
                               │                            │
                               └────────────────────────────┘
                                             │
                                   ┌─────────▼─────────┐
                                   │ skillhub-storage   │
                                   │  对象存储抽象       │
                                   └───────────────────┘
```

### 2.2 各模块职责

| 模块 | 职责 | 类数量 | 关键依赖 |
|------|------|--------|---------|
| **skillhub-storage** | 统一对象存储抽象（本地文件 / S3） | ~7 | AWS S3 SDK |
| **skillhub-domain** | 核心领域模型、领域服务、仓储接口、领域事件 | ~100+ | storage |
| **skillhub-notification** | 站内通知、通知偏好、SSE 实时推送 | ~12 | domain |
| **skillhub-auth** | OAuth2 登录、本地认证、设备码流、API Token、RBAC、账号合并 | ~55 | domain |
| **skillhub-infra** | JPA Repository 实现、HTTP 客户端、扫描器集成 | ~37 | domain, notification |
| **skillhub-search** | 全文搜索（PostgreSQL）、中文分词（jieba）、索引管理 | ~17 | domain, infra |
| **skillhub-app** | REST Controller、DTO、配置、兼容层、数据库迁移（41个版本） | ~160+ | 所有模块 |

### 2.3 skillhub-domain 核心领域子包

```
com.iflytek.skillhub.domain
├── audit/           # 审计日志（AuditLog, AuditLogService）
├── auth/            # 密码重置（PasswordResetRequest）
├── event/           # 领域事件（12个事件：发布、下载、审核、晋升、举报...）
├── governance/      # 治理通知（UserNotification, GovernanceNotificationService）
├── idempotency/     # 幂等性控制（IdempotencyRecord）
├── label/           # 标签系统（LabelDefinition, SkillLabel, 多语言翻译）
├── namespace/       # 命名空间（Namespace, 成员管理, 角色权限, 访问策略）
├── report/          # 举报（SkillReport, 举报处理）
├── review/          # 审核（ReviewTask, PromotionRequest, 审核/晋升服务）
├── security/        # 安全审计（SecurityAudit, 扫描任务, 判定结果）
├── skill/           # 技能核心（Skill, SkillVersion, SkillFile, 技能标签, 下载统计）
│   ├── metadata/        # 技能包元数据解析
│   ├── service/         # 技能服务（发布、查询、下载、治理、生命周期、slug解析）
│   └── validation/      # 发布前置校验（包校验、扩展名白名单、策略）
├── social/          # 社交功能（评分、收藏、订阅）
└── user/            # 用户（UserAccount, 资料审核、资料变更管理）
```

### 2.4 skillhub-app Controller 分组

```
controller/
├── AuthController, LocalAuthController, DeviceAuthController  # 认证
├── TokenController, UserProfileController, AccountMergeController  # 用户/Token
├── HealthController  # 健康检查
├── admin/            # 管理员接口（用户管理、审计、标签、资料审核、搜索重建）
├── cli/              # CLI 专用接口（认证、技能CRUD）
├── portal/           # Web 门户接口（技能搜索、发布、审核、晋升、命名空间、通知、评分、收藏...）
└── compat/           # ClawHub 兼容层（Canonical Slug 解析、OpenClaw 协议适配）
```

### 2.5 关键配置文件

| 文件 | 路径 |
|------|------|
| 主配置 | `server/skillhub-app/src/main/resources/application.yml` |
| 本地开发 | `server/skillhub-app/src/main/resources/application-local.yml` |
| 国际化(中) | `server/skillhub-app/src/main/resources/messages_zh.properties` |
| 数据库迁移 | `server/skillhub-app/src/main/resources/db/migration/` (V1~V41) |
| 限流脚本 | `server/skillhub-app/src/main/resources/ratelimit.lua` |

---

## 三、前端模块架构（web/）

采用 **Feature-Sliced Design** 变体，14 个业务功能模块。

### 3.1 技术栈

| 层面 | 技术 |
|------|------|
| 框架 | React 19 + TypeScript 5.7 |
| 构建 | Vite 6.1 |
| 路由 | @tanstack/react-router（懒加载 + 认证守卫） |
| 数据获取 | @tanstack/react-query v5（30s staleTime） |
| 客户端状态 | zustand v5 |
| API 类型 | openapi-typescript 自动生成 |
| UI 组件 | Tailwind CSS + Radix UI + CVA |
| 国际化 | i18next（中/英） |
| 实时通知 | SSE（Server-Sent Events） |

### 3.2 目录结构

```
web/src/
├── api/                     # API 客户端层
│   ├── client.ts            # 集中式 API 函数（10 个 API 模块）
│   ├── types.ts             # 前端业务类型扩展
│   └── generated/
│       └── schema.d.ts      # OpenAPI 自动生成类型
│
├── app/                     # 应用骨架
│   ├── layout.tsx           # 全局布局（Header + Footer + Outlet）
│   ├── providers.tsx        # Provider（QueryClient + Router + Toaster）
│   └── router.tsx           # 路由注册（公开/认证/角色保护）
│
├── features/                # 业务功能模块（14 个）
│   ├── admin/               # 管理员（用户管理、标签、审计日志）
│   ├── auth/                # 认证（登录、注册、OAuth、会话引导）
│   ├── governance/          # 治理（收件箱、活动流）
│   ├── namespace/           # 命名空间（CRUD、成员管理、所有权转移）
│   ├── notification/        # 通知（SSE 推送、未读缓存）
│   ├── promotion/           # 晋升管理
│   ├── publish/             # 技能发布（上传、预填充）
│   ├── report/              # 举报
│   ├── review/              # 审核（技能审核、文件查看）
│   ├── search/              # 搜索
│   ├── security-audit/      # 安全审计（发现项、严重性、裁决）
│   ├── skill/               # 技能核心（详情、版本、文件预览、Markdown 渲染）
│   ├── social/              # 社交（评分、收藏、订阅）
│   └── token/               # API Token 管理
│
├── pages/                   # 路由页面组件（全部懒加载）
├── shared/                  # 共享基础设施
│   ├── components/          # 业务级共享组件（分页、骨架屏、角色守卫...）
│   ├── hooks/               # 通用 Hooks（防抖、视口检测、查询辅助）
│   ├── lib/                 # 纯逻辑工具（错误处理、日期格式化、生命周期状态机）
│   └── ui/                  # 原子 UI 组件（Button、Card、Dialog...）
│
├── i18n/                    # 国际化配置与翻译文件
└── bootstrap.ts             # 运行时配置引导
```

### 3.3 API 模块列表

| API 模块 | 功能 |
|---------|------|
| `authApi` | 认证（登录、登出、当前用户、会话引导、直接登录） |
| `skillLifecycleApi` | 技能生命周期（归档、删除、撤回、重新发布、提交审核） |
| `namespaceApi` | 命名空间 CRUD、成员管理、所有权转移 |
| `reviewApi` | 审核列表、详情、批准/拒绝 |
| `promotionApi` | 晋升提交、列表、批准/拒绝 |
| `labelApi` | 标签管理（CRUD、关联/取消技能标签） |
| `governanceApi` | 治理（摘要、收件箱、活动流、通知） |
| `meApi` | 我的技能、收藏、订阅 |
| `adminApi` | 管理员操作（用户管理、审计日志、技能下架） |
| `notificationApi` | 通知（列表、标记已读、偏好设置） |

### 3.4 路由结构

**公开页面**: `/` (首页)、`/skills` (技能列表)、`/search`、`/login`、`/register`、`/space/:namespace/:slug` (技能详情)

**认证页面**: `/dashboard` (仪表盘)、`/dashboard/skills`、`/dashboard/publish`、`/dashboard/namespaces`、`/dashboard/reviews`、`/dashboard/tokens`

**角色保护页面**: `/admin/users`（USER_ADMIN）、`/admin/audit-log`（AUDITOR）、`/admin/labels`（SUPER_ADMIN）

---

## 四、CLI 模块架构（cli/）

采用 **四层分层架构**，基于 Bun 运行时。

### 4.1 技术栈

| 层面 | 技术 |
|------|------|
| 语言/运行时 | TypeScript / Bun |
| CLI 框架 | cac（命令解析） |
| 交互提示 | prompts |
| ZIP 压缩 | fflate（纯 JS，含路径遍历防护） |
| 版本管理 | semver |

### 4.2 目录结构

```
cli/src/
├── index.ts                  # 入口（命令注册、模糊匹配、统一错误处理）
├── commands/                 # 命令层（12 个命令）
│   ├── login, logout, whoami # 认证
│   ├── search, install, list, remove  # 技能管理
│   ├── publish               # 技能发布
│   ├── doctor                # 清单修复
│   ├── update                # CLI 自更新
│   └── help, version         # 辅助
├── services/                 # 服务层（核心业务逻辑）
├── clients/                  # 客户端层
│   ├── skillhub-client.ts    # SkillHub REST API 客户端
│   └── npm-registry-client.ts # npm 版本查询
├── agents/                   # Agent 适配层（14 种 AI 工具）
│   ├── detector.ts           # Agent 检测器
│   ├── resolver.ts           # 安装目标解析器
│   └── profiles/             # 各 Agent 配置（Claude/Cursor/Copilot/Gemini...）
├── stores/                   # 持久化层（~/.skillhub/）
│   ├── config-store.ts       # 全局配置
│   ├── inventory-store.ts    # 安装清单（原子写入 + 文件锁）
│   └── credentials-store.ts  # 凭据存储
├── platform/                 # 平台抽象（OS 检测、路径工具、子进程执行）
└── shared/                   # 基础设施（常量、类型、错误类、输出格式化）
```

### 4.3 支持的 AI Agent（14 种）

| Agent | 安装目录 |
|-------|---------|
| Claude Code | `.claude/skills` |
| Cursor | 对应 skills 目录 |
| GitHub Copilot | 对应 skills 目录 |
| Gemini CLI | 对应 skills 目录 |
| Windsurf | 对应 skills 目录 |
| OpenHands | 对应 skills 目录 |
| OpenClaw | 对应 skills 目录 |
| Kiro / Roo / Trae / OpenCode / Kilo | 对应 skills 目录 |
| 通用回退 | `.agents/skills` |

### 4.4 配置/令牌解析优先级

```
--registry 参数 > SKILLHUB_REGISTRY 环境变量 > config.json > 默认值
--token 参数    > SKILLHUB_TOKEN 环境变量    > credentials.json
```

---

## 五、安全扫描器模块（scanner/）

基于 Cisco 开源 `cisco-ai-skill-scanner` 的 Docker 化封装。

### 5.1 扫描引擎

| 分析器 | 功能 | 状态 |
|--------|------|------|
| Meta | 包元数据、依赖、许可证检查 | 默认启用 |
| 静态规则（Regex + YARA） | 模式匹配，13 个内置 YARA 文件 | 默认启用 |
| Behavioral | 可疑运行时行为检测 | 可选 |
| LLM | 大语言模型语义分析 | 可选 |
| VirusTotal | 已知恶意文件检测 | 可选 |

### 5.2 扫描集成流程

```
后端发布技能
  → SecurityScanService 创建 SecurityAudit
  → 发送消息到 Redis Stream (skillhub:scan:requests)
  → ScanTaskConsumer 消费
  → SkillScannerService HTTP 调用 Scanner API (:8000/scan-upload)
  → 结果写入 SecurityAudit（成功/失败自动降级到人工审核）
```

---

## 六、基础设施与部署

### 6.1 Docker Compose 服务拓扑

```
              ┌──────┐
              │ 用户 │
              └──┬───┘
                 │
          ┌──────▼──────┐
          │   Nginx     │  :80  反向代理
          │   (web)     │──────┐
          └──────┬──────┘      │
                 │ /api         │
          ┌──────▼──────┐      │
          │  Spring Boot│      │
          │  (server)   │      │
          │   :8080     │      │
          └──┬───┬──┬───┘      │
             │   │  │          │
      ┌──────┘   │  └───────┐  │
      ▼          ▼          ▼  ▼
  PostgreSQL  Redis     Scanner  S3/MinIO
   :5432     :6379      :8000   :9000
```

### 6.2 三套 Compose 环境

| 环境 | 文件 | 特点 |
|------|------|------|
| 开发 | `docker-compose.yml` | 仅依赖服务（scanner + pg + redis + minio），应用本地运行 |
| Staging | `+ docker-compose.staging.yml` | 混合模式：应用本地构建镜像 |
| 发布 | `compose.release.yml` | 全部从 GHCR 拉取预构建镜像 |

### 6.3 K8s 部署（Kustomize）

```
deploy/k8s/
├── base/                    # 基础配置
│   ├── backend-deployment   # 1 副本
│   ├── frontend-deployment  # 1 副本
│   ├── scanner-deployment   # 1 副本
│   ├── ingress.yaml         # Nginx Ingress
│   └── configmap.yaml       # 非敏感配置
└── overlays/
    ├── with-infra/          # 含内置 Postgres + Redis StatefulSet
    └── external/            # 使用外部数据库
```

部署命令：
```bash
kubectl apply -k deploy/k8s/overlays/with-infra/   # 完整部署
kubectl apply -k deploy/k8s/overlays/external/       # 外部数据库
```

### 6.4 监控

| 组件 | 端口 | 功能 |
|------|------|------|
| Prometheus | 9090 | 指标采集（15s 间隔，后端 `/actuator/prometheus`） |
| Grafana | 3001 | 可视化面板（默认 admin/admin） |

---

## 七、核心数据流

### 7.1 技能发布流程

```
用户上传 ZIP → SkillPublishController
  → PrePublishValidator（扩展名白名单 + 策略校验）
  → SkillPublishService（解析元数据、创建版本）
  → 触发 SkillPublishedEvent
    → SearchIndexEventListener（更新搜索索引）
    → ScanTaskProducer（发送到 Redis Stream）
      → Scanner 异步扫描
        → 成功：版本状态流转
        → 失败：自动创建 ReviewTask 降级到人工审核
```

### 7.2 技能搜索流程

```
用户搜索 → SkillSearchController
  → SearchQueryService（PostgreSQL 全文搜索 + jieba 中文分词）
  → VisibilityChecker（按用户权限过滤可见结果）
  → 返回 SearchResult 列表
```

### 7.3 认证流程

```
Web 登录 → OAuth2（GitHub/GitLab）或本地密码
  → OAuthLoginFlowService / LocalAuthService
  → IdentityBindingService（身份绑定/账号合并）
  → PlatformSessionService（Redis Session）
  → RbacService（角色权限判断）
  → Controller 方法级权限校验
```

---

## 八、常用开发命令

```bash
# 一键启动完整开发环境
make dev-all

# 构建前后端
make build

# 运行测试
make test

# 类型检查
make typecheck-web

# 生成 OpenAPI 类型
make generate-api

# 冒烟测试
./scripts/smoke-test.sh http://localhost:8080

# Staging 环境
make staging

# CLI 构建
make build-cli
```
