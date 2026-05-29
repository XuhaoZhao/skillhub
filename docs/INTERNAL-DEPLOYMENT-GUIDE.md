# SkillHub 内网部署指南

> 本文档面向需要在公司内网（离线/半离线环境）部署 SkillHub 的运维人员。

---

## 一、你需要什么——整体感知

### 1.1 服务全景图

部署 SkillHub 实际上需要运行以下服务：

```
                    ┌──────────────────────────────────────────┐
                    │           用户浏览器 / CLI                 │
                    └──────────────┬───────────────────────────┘
                                   │ :80
                    ┌──────────────▼───────────────────────────┐
                    │         web（Nginx 静态前端）              │  ← 镜像: skillhub-web
                    │         负责: 页面渲染 + API 反向代理       │
                    └──────────────┬───────────────────────────┘
                                   │ /api → 反向代理
                    ┌──────────────▼───────────────────────────┐
                    │       server（Spring Boot 后端）          │  ← 镜像: skillhub-server
                    │       负责: 业务逻辑、REST API             │
                    └──┬─────────┬─────────────┬──────────────┘
                       │         │             │
              ┌────────▼──┐ ┌───▼───────┐ ┌───▼──────────────┐
              │ PostgreSQL│ │   Redis    │ │ skill-scanner     │  ← 镜像: skillhub-scanner（可选）
              │  数据库    │ │  缓存/会话  │ │ 安全扫描器（Python）│
              └───────────┘ └───────────┘ └──────────────────┘
              必需           必需            可选
              (Docker 部署)   (你已有)         (可禁用)
```

### 1.2 资源清单

| 类别 | 资源 | 来源 | 说明 |
|------|------|------|------|
| **Docker 镜像** | `ghcr.io/iflytek/skillhub-server` | GHCR | 后端 Java 服务 |
| **Docker 镜像** | `ghcr.io/iflytek/skillhub-web` | GHCR | 前端 Nginx 服务 |
| **Docker 镜像** | `ghcr.io/iflytek/skillhub-scanner` | GHCR | 安全扫描器（可选） |
| **Docker 镜像** | `postgres:16-alpine` | Docker Hub | PostgreSQL 数据库 |
| **基础设施** | Redis | **你公司已有** | 用于 Session 和异步消息 |
| **存储** | S3/MinIO 或本地磁盘 | 你公司提供 | 技能包文件存储 |
| **文件** | `compose.release.yml` | 本仓库 | Docker Compose 编排文件 |
| **文件** | `.env.release` | 需要创建 | 环境变量配置 |

### 1.3 存储方案选择

| 方案 | 适用场景 | 配置 |
|------|---------|------|
| **本地磁盘**（Docker Volume） | 评估测试、小规模使用 | `SKILLHUB_STORAGE_PROVIDER=local` |
| **公司 S3/MinIO** | 生产环境、需要持久化和共享 | `SKILLHUB_STORAGE_PROVIDER=s3` + 配置 S3 参数 |
| **公司已有 MinIO** | 公司内已有 MinIO 实例 | 填写已有 MinIO 的 Endpoint 和凭证 |

---

## 二、部署步骤

### 第一步：在外网机器上准备镜像

找一台能访问外网（GHCR 和 Docker Hub）的机器，执行以下操作。

#### 1.1 拉取所有镜像

```bash
# 指定版本号（推荐用 latest，或指定如 v0.2.0）
VERSION=latest

# 拉取 SkillHub 三个镜像
docker pull ghcr.io/iflytek/skillhub-server:${VERSION}
docker pull ghcr.io/iflytek/skillhub-web:${VERSION}
docker pull ghcr.io/iflytek/skillhub-scanner:${VERSION}

# 拉取 PostgreSQL
docker pull postgres:16-alpine
```

> **国内用户**：如果 GHCR 拉取慢，可尝试配置 Docker 代理，或使用阿里云镜像（参考 `deploy/runtime-mirror-images.txt` 中的镜像映射）。

#### 1.2 导出镜像为 tar 文件

```bash
# 统一导出为一个 tar 包（方便传输）
docker save \
  ghcr.io/iflytek/skillhub-server:${VERSION} \
  ghcr.io/iflytek/skillhub-web:${VERSION} \
  ghcr.io/iflytek/skillhub-scanner:${VERSION} \
  postgres:16-alpine \
  -o skillhub-images-${VERSION}.tar
```

文件大约 **1.5~2 GB**（取决于版本）。

#### 1.3 传输到内网

```bash
# 通过 scp、U 盘、内网文件服务器等方式传输到内网目标机器
scp skillhub-images-${VERSION}.tar user@internal-server:/tmp/
```

---

### 第二步：在内网目标机器上加载镜像

```bash
# 加载镜像（可能需要几分钟）
docker load -i /tmp/skillhub-images-${VERSION}.tar

# 验证镜像已加载
docker images | grep skillhub
# 预期输出：
# ghcr.io/iflytek/skillhub-server   latest   ...
# ghcr.io/iflytek/skillhub-web      latest   ...
# ghcr.io/iflytek/skillhub-scanner  latest   ...
# postgres                          16-alpine ...
```

---

### 第三步：准备项目文件

在内网目标机器上，你需要两个文件：

1. **`compose.release.yml`** —— 从仓库复制
2. **`.env.release`** —— 你需要创建的环境变量配置

```bash
mkdir -p /opt/skillhub && cd /opt/skillhub

# 从仓库复制 compose 文件（如果已有仓库代码）
cp /path/to/skillhub/compose.release.yml .

# 或者手动创建（内容见下方）
```

---

### 第四步：创建环境变量配置文件

创建 `.env.release` 文件。以下是针对你的场景（PostgreSQL 用 Docker 部署、Redis 用公司已有的）的配置模板：

```bash
cat > /opt/skillhub/.env.release << 'EOF'
# ============================================================
#  SkillHub 内网部署配置
# ============================================================

# ---- 版本与镜像 ----
# 指定本地已加载的镜像版本（不设则默认 latest）
SKILLHUB_VERSION=latest

# ============================================================
#  必填配置
# ============================================================

# 公网访问地址（重要！影响 CLI 安装命令和 OAuth 回调地址）
# 示例: http://skillhub.your-company.com 或 https://skillhub.your-company.com
SKILLHUB_PUBLIC_BASE_URL=http://skillhub.your-company.com

# PostgreSQL 配置（Docker 内置）
POSTGRES_DB=skillhub
POSTGRES_USER=skillhub
POSTGRES_PASSWORD=<改成强密码，如: Sk1llHub@2026!Pg>
POSTGRES_PORT=5432
POSTGRES_BIND_ADDRESS=127.0.0.1

# ---- Redis 配置（连接公司已有 Redis） ----
# 注意：compose.release.yml 默认会启动 redis 容器，
#       你需要改为指向公司已有 Redis，见下方第六步的自定义 compose 文件
# Redis 连接信息（填写公司内网 Redis 的地址）
REDIS_HOST=<公司Redis的IP或域名，如: 10.0.1.100>
REDIS_PORT=6379
REDIS_PASSWORD=<公司Redis密码，无密码则留空>

# ---- 存储配置 ----
# 方案 A：本地存储（简单，数据在 Docker Volume 中）
SKILLHUB_STORAGE_PROVIDER=local

# 方案 B：公司 MinIO/S3（推荐生产使用）
# SKILLHUB_STORAGE_PROVIDER=s3
# SKILLHUB_STORAGE_S3_ENDPOINT=http://<MinIO地址>:9000
# SKILLHUB_STORAGE_S3_PUBLIC_ENDPOINT=http://<MinIO地址>:9000
# SKILLHUB_STORAGE_S3_BUCKET=skillhub
# SKILLHUB_STORAGE_S3_ACCESS_KEY=<AccessKey>
# SKILLHUB_STORAGE_S3_SECRET_KEY=<SecretKey>
# SKILLHUB_STORAGE_S3_REGION=us-east-1
# SKILLHUB_STORAGE_S3_FORCE_PATH_STYLE=true
# SKILLHUB_STORAGE_S3_AUTO_CREATE_BUCKET=true

# ============================================================
#  管理员账号
# ============================================================
BOOTSTRAP_ADMIN_ENABLED=true
BOOTSTRAP_ADMIN_USER_ID=docker-admin
BOOTSTRAP_ADMIN_USERNAME=admin
BOOTSTRAP_ADMIN_PASSWORD=<改成强密码>
BOOTSTRAP_ADMIN_DISPLAY_NAME=SkillHub管理员
BOOTSTRAP_ADMIN_EMAIL=admin@your-company.com

# ============================================================
#  认证方式
# ============================================================

# 方式 A：直接登录（推荐内网使用，用户名+密码直接登录）
SKILLHUB_AUTH_DIRECT_ENABLED=true

# 方式 B：OAuth2（如果公司有 GitHub/GitLab，可配置）
# OAUTH2_GITHUB_CLIENT_ID=<your-github-client-id>
# OAUTH2_GITHUB_CLIENT_SECRET=<your-github-client-secret>

# 前端直接登录开关（配合后端 SKILLHUB_AUTH_DIRECT_ENABLED）
SKILLHUB_WEB_AUTH_DIRECT_ENABLED=true

# ============================================================
#  安全扫描器
# ============================================================
# 设为 false 可跳过扫描（降低部署复杂度）
# 设为 true 需要部署 scanner 容器
SKILLHUB_SECURITY_SCANNER_ENABLED=true

# ============================================================
#  网络与端口
# ============================================================
WEB_PORT=80
API_PORT=8080

# Session 配置（HTTP 环境设 false，HTTPS 设 true）
SESSION_COOKIE_SECURE=false

# 前端 API 地址（通常留空，由 Nginx 反向代理处理）
SKILLHUB_WEB_API_BASE_URL=

# ============================================================
#  邮件（可选，用于密码重置）
# ============================================================
# SPRING_MAIL_HOST=smtp.your-company.com
# SPRING_MAIL_PORT=25
# SPRING_MAIL_USERNAME=
# SPRING_MAIL_PASSWORD=
EOF
```

> **重要**：请把 `<改成强密码>` 等占位符替换为实际值。

---

### 第五步：创建自定义 compose 文件

由于你要**复用公司已有 Redis** 而不启动内置 Redis 容器，需要基于 `compose.release.yml` 创建一个自定义版本。

创建 `/opt/skillhub/docker-compose.custom.yml`：

```yaml
services:
  # ---- 安全扫描器（可选） ----
  skill-scanner:
    image: ${SKILLHUB_SCANNER_IMAGE:-ghcr.io/iflytek/skillhub-scanner}:${SKILLHUB_VERSION:-latest}
    restart: unless-stopped
    profiles:
      - scanner    # 使用 profile 控制，需要时才启动
    environment:
      SKILL_SCANNER_LLM_API_KEY: ${SKILL_SCANNER_LLM_API_KEY:-}
      SKILL_SCANNER_LLM_BASE_URL: ${SKILL_SCANNER_LLM_BASE_URL:-}
      SKILL_SCANNER_LLM_MODEL: ${SKILL_SCANNER_LLM_MODEL:-}
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 10

  # ---- PostgreSQL（Docker 内置） ----
  postgres:
    image: ${POSTGRES_IMAGE:-postgres:16-alpine}
    restart: unless-stopped
    ports:
      - "${POSTGRES_BIND_ADDRESS:-127.0.0.1}:${POSTGRES_PORT:-5432}:5432"
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-skillhub}
      POSTGRES_USER: ${POSTGRES_USER:-skillhub}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-skillhub_demo}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-skillhub} -d ${POSTGRES_DB:-skillhub}"]
      interval: 5s
      timeout: 5s
      retries: 10

  # ---- 注意：没有 redis 服务，使用公司已有 Redis ----

  # ---- 后端服务 ----
  server:
    image: ${SKILLHUB_SERVER_IMAGE:-ghcr.io/iflytek/skillhub-server}:${SKILLHUB_VERSION:-latest}
    restart: unless-stopped
    ports:
      - "${API_PORT:-8080}:8080"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/${POSTGRES_DB:-skillhub}
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER:-skillhub}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD:-skillhub_demo}
      # ---- Redis 指向公司已有实例 ----
      REDIS_HOST: ${REDIS_HOST:?请设置REDIS_HOST}
      REDIS_PORT: ${REDIS_PORT:-6379}
      REDIS_PASSWORD: ${REDIS_PASSWORD:-}
      SPRING_DATA_REDIS_PASSWORD: ${REDIS_PASSWORD:-}
      # ---- Session 安全 ----
      SESSION_COOKIE_SECURE: ${SESSION_COOKIE_SECURE:-false}
      # ---- 公网地址 ----
      SKILLHUB_PUBLIC_BASE_URL: ${SKILLHUB_PUBLIC_BASE_URL:-}
      DEVICE_AUTH_VERIFICATION_URI: ${DEVICE_AUTH_VERIFICATION_URI:-}
      # ---- 存储 ----
      SKILLHUB_STORAGE_PROVIDER: ${SKILLHUB_STORAGE_PROVIDER:-local}
      STORAGE_BASE_PATH: /var/lib/skillhub/storage
      SKILLHUB_STORAGE_S3_ENDPOINT: ${SKILLHUB_STORAGE_S3_ENDPOINT:-}
      SKILLHUB_STORAGE_S3_PUBLIC_ENDPOINT: ${SKILLHUB_STORAGE_S3_PUBLIC_ENDPOINT:-}
      SKILLHUB_STORAGE_S3_BUCKET: ${SKILLHUB_STORAGE_S3_BUCKET:-skillhub}
      SKILLHUB_STORAGE_S3_ACCESS_KEY: ${SKILLHUB_STORAGE_S3_ACCESS_KEY:-}
      SKILLHUB_STORAGE_S3_SECRET_KEY: ${SKILLHUB_STORAGE_S3_SECRET_KEY:-}
      SKILLHUB_STORAGE_S3_REGION: ${SKILLHUB_STORAGE_S3_REGION:-us-east-1}
      SKILLHUB_STORAGE_S3_FORCE_PATH_STYLE: ${SKILLHUB_STORAGE_S3_FORCE_PATH_STYLE:-false}
      SKILLHUB_STORAGE_S3_AUTO_CREATE_BUCKET: ${SKILLHUB_STORAGE_S3_AUTO_CREATE_BUCKET:-false}
      SKILLHUB_STORAGE_S3_PRESIGN_EXPIRY: ${SKILLHUB_STORAGE_S3_PRESIGN_EXPIRY:-PT10M}
      # ---- 安全扫描器 ----
      SKILLHUB_SECURITY_SCANNER_ENABLED: ${SKILLHUB_SECURITY_SCANNER_ENABLED:-true}
      SKILLHUB_SECURITY_SCANNER_URL: ${SKILLHUB_SECURITY_SCANNER_URL:-http://skill-scanner:8000}
      SKILLHUB_SECURITY_SCANNER_MODE: ${SKILLHUB_SECURITY_SCANNER_MODE:-upload}
      # ---- 认证 ----
      SKILLHUB_AUTH_DIRECT_ENABLED: ${SKILLHUB_AUTH_DIRECT_ENABLED:-false}
      # ---- 管理员 ----
      BOOTSTRAP_ADMIN_ENABLED: ${BOOTSTRAP_ADMIN_ENABLED:-false}
      BOOTSTRAP_ADMIN_USER_ID: ${BOOTSTRAP_ADMIN_USER_ID:-docker-admin}
      BOOTSTRAP_ADMIN_USERNAME: ${BOOTSTRAP_ADMIN_USERNAME:-admin}
      BOOTSTRAP_ADMIN_PASSWORD: ${BOOTSTRAP_ADMIN_PASSWORD:-ChangeMe!2026}
      BOOTSTRAP_ADMIN_DISPLAY_NAME: ${BOOTSTRAP_ADMIN_DISPLAY_NAME:-Admin}
      BOOTSTRAP_ADMIN_EMAIL: ${BOOTSTRAP_ADMIN_EMAIL:-admin@skillhub.local}
      # ---- OAuth2（如不使用可保持默认占位符） ----
      OAUTH2_GITHUB_CLIENT_ID: ${OAUTH2_GITHUB_CLIENT_ID:-local-placeholder}
      OAUTH2_GITHUB_CLIENT_SECRET: ${OAUTH2_GITHUB_CLIENT_SECRET:-local-placeholder}
      # ---- 邮件 ----
      SPRING_MAIL_HOST: ${SPRING_MAIL_HOST:-}
      SPRING_MAIL_PORT: ${SPRING_MAIL_PORT:-25}
      SPRING_MAIL_USERNAME: ${SPRING_MAIL_USERNAME:-}
      SPRING_MAIL_PASSWORD: ${SPRING_MAIL_PASSWORD:-}
      SPRING_MAIL_SMTP_AUTH: ${SPRING_MAIL_SMTP_AUTH:-false}
      SPRING_MAIL_SMTP_STARTTLS_ENABLE: ${SPRING_MAIL_SMTP_STARTTLS_ENABLE:-false}
      SPRING_MAIL_PROPERTIES_MAIL_SMTP_SSL_ENABLE: ${SPRING_MAIL_PROPERTIES_MAIL_SMTP_SSL_ENABLE:-false}
      SPRING_MAIL_PROPERTIES_MAIL_SMTP_SSL_TRUST: ${SPRING_MAIL_PROPERTIES_MAIL_SMTP_SSL_TRUST:-}
    volumes:
      - skillhub_storage:/var/lib/skillhub/storage
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 60s

  # ---- 前端服务 ----
  web:
    image: ${SKILLHUB_WEB_IMAGE:-ghcr.io/iflytek/skillhub-web}:${SKILLHUB_VERSION:-latest}
    restart: unless-stopped
    ports:
      - "${WEB_PORT:-80}:80"
    environment:
      SKILLHUB_API_UPSTREAM: ${SKILLHUB_API_UPSTREAM:-http://server:8080}
      SKILLHUB_WEB_API_BASE_URL: ${SKILLHUB_WEB_API_BASE_URL:-}
      SKILLHUB_PUBLIC_BASE_URL: ${SKILLHUB_PUBLIC_BASE_URL:-}
      SKILLHUB_WEB_AUTH_DIRECT_ENABLED: ${SKILLHUB_WEB_AUTH_DIRECT_ENABLED:-false}
      SKILLHUB_WEB_AUTH_DIRECT_PROVIDER: ${SKILLHUB_WEB_AUTH_DIRECT_PROVIDER:-}
    depends_on:
      server:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1/nginx-health"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 10s

volumes:
  postgres_data:
  # 注意：没有 redis_data，使用公司已有 Redis
  skillhub_storage:
```

> **关键差异说明**：
> - 移除了 `redis` 服务定义和 `redis_data` 卷
> - `server` 不再 `depends_on redis`（因为 Redis 不在本 compose 内，你需确保 Redis 可达）
> - scanner 使用 `profiles: [scanner]`，需要时通过 `--profile scanner` 启动

---

### 第六步：启动服务

```bash
cd /opt/skillhub

# 加载环境变量
export $(grep -v '^#' .env.release | grep -v '^$' | xargs)

# 启动基础服务（postgres + server + web）
docker compose -f docker-compose.custom.yml up -d

# 如果需要安全扫描器，加上 scanner profile
# docker compose -f docker-compose.custom.yml --profile scanner up -d
```

> **注意**：如果禁用了扫描器（`SKILLHUB_SECURITY_SCANNER_ENABLED=false`），则不需要 scanner 容器，直接启动即可。

---

### 第七步：验证部署

```bash
# 1. 检查所有容器状态
docker compose -f docker-compose.custom.yml ps

# 2. 检查后端健康
curl http://localhost:8080/actuator/health
# 预期: {"status":"UP",...}

# 3. 检查前端
curl http://localhost/nginx-health
# 预期: ok

# 4. 浏览器访问
# http://<服务器IP>:80
```

如果启用了管理员账号，使用以下凭据登录：
- 用户名：你在 `BOOTSTRAP_ADMIN_USERNAME` 中设置的值（默认 `admin`）
- 密码：你在 `BOOTSTRAP_ADMIN_PASSWORD` 中设置的值

---

## 三、常见问题

### 3.1 如何配置公司已有 MinIO 作为存储？

在 `.env.release` 中设置：

```bash
SKILLHUB_STORAGE_PROVIDER=s3
SKILLHUB_STORAGE_S3_ENDPOINT=http://<MinIO内网IP>:9000
SKILLHUB_STORAGE_S3_PUBLIC_ENDPOINT=http://<MinIO内网IP>:9000
SKILLHUB_STORAGE_S3_BUCKET=skillhub
SKILLHUB_STORAGE_S3_ACCESS_KEY=<你的AccessKey>
SKILLHUB_STORAGE_S3_SECRET_KEY=<你的SecretKey>
SKILLHUB_STORAGE_S3_REGION=us-east-1
SKILLHUB_STORAGE_S3_FORCE_PATH_STYLE=true
SKILLHUB_STORAGE_S3_AUTO_CREATE_BUCKET=true
```

### 3.2 如何配置公司 GitLab OAuth2 登录？

在 `.env.release` 中设置（注意：GitLab 需要内网可达）：

```bash
OAUTH2_GITLAB_CLIENT_ID=<GitLab Application ID>
OAUTH2_GITLAB_CLIENT_SECRET=<GitLab Secret>
OAUTH2_GITLAB_BASE_URI=https://gitlab.your-company.com
OAUTH2_GITLAB_DISPLAY_NAME=公司GitLab
```

### 3.3 如何使用 HTTPS？

建议在 `web` 前面放一层 Nginx 反向代理（或公司已有网关），负责 SSL 终止：

```bash
# 在 .env.release 中设置
SESSION_COOKIE_SECURE=true
SKILLHUB_PUBLIC_BASE_URL=https://skillhub.your-company.com
```

### 3.4 如何禁用安全扫描器（简化部署）？

```bash
# .env.release
SKILLHUB_SECURITY_SCANNER_ENABLED=false
```

这样就不需要 scanner 容器，也不用担心 scanner 镜像的 pip 依赖需要外网。

### 3.5 如果公司 Redis 有密码？

确保在 `.env.release` 和 compose 文件中都设置了：

```bash
REDIS_PASSWORD=<你的Redis密码>
```

compose 文件中后端需要两个环境变量：
```yaml
REDIS_PASSWORD: ${REDIS_PASSWORD:-}
SPRING_DATA_REDIS_PASSWORD: ${REDIS_PASSWORD:-}
```

### 3.6 如果公司 Redis 不可达导致启动失败？

检查以下几点：
1. 防火墙是否放通 Redis 端口（默认 6379）
2. Redis 是否配置了 `bind` 限制（需要允许目标机器的 IP 访问）
3. 是否需要在 Redis 配置中设置 `protected-mode no` 或配置 `requirepass`

### 3.7 数据库迁移会自动执行吗？

是的。后端启动时会自动运行 Flyway 迁移（`db/migration/` 目录下 V1~V41），无需手动初始化数据库。只需确保 PostgreSQL 服务正常且连接信息正确。

---

## 四、服务器硬件建议

| 规模 | CPU | 内存 | 磁盘 |
|------|-----|------|------|
| **评估测试** | 2 核 | 4 GB | 20 GB SSD |
| **小团队（<50人）** | 4 核 | 8 GB | 100 GB SSD |
| **中等规模（50~200人）** | 8 核 | 16 GB | 500 GB SSD |

> 说明：PostgreSQL 约 1~2 GB 内存，Java 后端（JVM）建议分配 2~4 GB（通过 `-XX:MaxRAMPercentage=75.0` 自动按容器内存 75% 计算）。

---

## 五、日常运维

```bash
cd /opt/skillhub

# 查看日志
docker compose -f docker-compose.custom.yml logs -f server

# 重启服务
docker compose -f docker-compose.custom.yml restart server

# 停止所有服务
docker compose -f docker-compose.custom.yml down

# 停止并清除数据卷（危险！会删数据）
docker compose -f docker-compose.custom.yml down -v

# 更新版本（重新加载镜像后）
docker compose -f docker-compose.custom.yml pull   # 如果有内网镜像仓库
docker compose -f docker-compose.custom.yml up -d
```

---

## 六、升级流程

```bash
# 1. 在外网机器拉取新版镜像
docker pull ghcr.io/iflytek/skillhub-server:<new-version>
docker pull ghcr.io/iflytek/skillhub-web:<new-version>
docker pull ghcr.io/iflytek/skillhub-scanner:<new-version>

# 2. 导出、传输、加载（同上）

# 3. 在内网目标机器上
cd /opt/skillhub
# 更新 .env.release 中的版本号
# SKILLHUB_VERSION=<new-version>

# 4. 重新启动
docker compose -f docker-compose.custom.yml up -d

# 5. 验证
curl http://localhost:8080/actuator/health
```

> **注意**：数据库迁移是自动的、向前兼容的。但建议升级前备份 PostgreSQL 数据：
> ```bash
> docker exec skillhub-postgres-1 pg_dump -U skillhub skillhub > backup.sql
> ```
