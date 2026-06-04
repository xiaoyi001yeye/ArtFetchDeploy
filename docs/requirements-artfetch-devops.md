# ArtFetchDeploy 需求文档

状态：草案  
更新日期：2026-06-05  
DevOps 项目目录：`/Users/wyn/code/ArtFetchDeploy`  
目标应用源码目录：`/Users/wyn/code/ArtFetch`

## 1. 背景

`ArtFetchDeploy` 是一个全新的 ArtFetch 本地 DevOps 项目，用于替代 ArtFetch 仓库内既有 GitHub Actions Release workflow 和部署脚本方案。ArtFetch 仓库只作为应用源码输入，不作为构建、发布、部署自动化的事实来源。

本项目从本地完成 ArtFetch 的预检、构建、离线制品打包、上传、服务器部署、健康检查、备份、回滚和审计。服务器只消费 ArtFetchDeploy 生成的离线发布包，不在服务器上构建源码、不拉取 registry 镜像。

## 2. 项目定位

`ArtFetchDeploy` 是 ArtFetch 的本地 DevOps 控制项目。它维护发布部署脚本、生产 Compose 模板、环境 profile、离线制品、部署报告和运维 runbook。

项目职责：

- 从 `/Users/wyn/code/ArtFetch` 读取源码、Dockerfile 和 Flyway 迁移。
- 在本地执行发布前检查。
- 在本地构建 backend、frontend、jupyter 运行镜像。
- 在本地拉取并导出运行所需外部镜像。
- 生成完整离线发布包和 `release-manifest.json`。
- 通过 SSH/SCP 上传发布包到服务器。
- 调用发布包内由 ArtFetchDeploy 生成的服务器脚本完成安装、升级、验证和清理。
- 执行升级前数据库备份和运行任务检查。
- 记录构建报告、部署报告、部署历史和回滚记录。

项目不做：

- 不依赖、不调用 `/Users/wyn/code/ArtFetch/.github/workflows/release.yml`。
- 不依赖、不调用 `/Users/wyn/code/ArtFetch/scripts/` 下任何发布或部署脚本。
- 不保存生产 `.env`、数据库 dump、雅昌 Cookie、对象存储密钥、SSH 私钥或 GitHub token。
- 不把核心部署逻辑写在 Jenkins Job 或其它外部 UI 配置中。
- 不在生产服务器执行 `npm run build`、`mvn package`、`docker build`、`docker pull` 或源码构建。

## 3. 已确认设计决策

1. 发布方式：本地构建完整离线包，通过 SSH/SCP 部署到服务器。
2. 自动化载体：脚本优先，Jenkins 延后；脚本是唯一事实来源。
3. ArtFetch 仓库角色：只提供源码、Dockerfile、Flyway 迁移和构建上下文。
4. Dockerfile：第一版复用 ArtFetch 现有 `backend/Dockerfile`、`frontend/Dockerfile`、`ml/Dockerfile`。
5. 生产 Compose：由 `ArtFetchDeploy/templates/docker-compose.prod.yml` 维护，作为生产运行形态事实来源。
6. 版本规则：发布版本必须等于 ArtFetch 最新 Flyway 迁移版本。
7. 源码洁净度：默认要求 ArtFetch 工作区干净；`--allow-dirty` 只允许临时或应急发布。
8. 构建范围：第一版全量构建、全量打包。
9. 发布分发：第一版不依赖 GitHub Release。
10. 服务器脚本：发布包内包含由 ArtFetchDeploy 生成的服务器安装、升级、验证、备份和清理脚本。
11. 数据库备份：已有部署升级前强制执行 `pg_dump -Fc`，备份失败停止部署。
12. 回滚：默认只回滚应用和 Compose；数据库恢复必须人工显式确认。
13. `.env`：服务器本地生成和保留，升级不覆盖。
14. 环境配置：第一版支持 profile，先落 `prod`，预留 `staging`。
15. 制品保留：本地和服务器默认保留最近 5 个成功版本。
16. 测试门禁：第一版强制构建通过，后端测试可选。
17. 外部镜像范围：发布包只包含运行时直接需要的镜像。
18. 运行任务检查：生产升级前硬门禁，发现运行或上传任务默认停止。
19. 部署验证：第一版只做容器和 HTTP 健康检查，不做登录级 smoke test。
20. 命令结构：拆成 preflight、build-release、deploy-release 三步，并提供组合入口。
21. sudo 策略：默认远程用户可直接运行 Docker，同时支持配置免密 sudo。

## 4. 总体目标

建立一套可重复、可审计、可恢复的本地发布部署链路，使发布负责人能够完成：

1. 检查 ArtFetch 源码、版本、迁移、构建环境和服务器环境。
2. 在本地构建完整离线发布包。
3. 校验发布包、manifest、镜像 tar 和 Compose 模板。
4. 上传指定版本到指定环境。
5. 在服务器执行安全安装或升级。
6. 升级前自动备份数据库并检查运行任务。
7. 部署后执行容器和 HTTP 健康检查。
8. 失败时保留数据，支持安全重试或回滚。

## 5. 目标环境

### 5.1 本地构建机

默认本地环境是开发者机器：

- DevOps 项目：`/Users/wyn/code/ArtFetchDeploy`
- ArtFetch 源码：`/Users/wyn/code/ArtFetch`
- 必需工具：
  - Git
  - Docker
  - Docker Compose plugin
  - Node.js
  - npm
  - Java
  - Maven
  - Python 3
  - SSH
  - SCP 或 rsync

本地构建机可以访问 Docker Hub、npm、Maven 等依赖源。生产服务器不需要访问这些依赖源。

### 5.2 生产服务器

默认服务器：

- SSH profile：`prod`
- 默认 SSH target：`artfetch-prod`
- 默认项目目录：`/opt/artfetch`
- 默认公开入口：`http://124.174.79.81:3000`

服务器必需工具：

- `bash`
- `tar`
- `sha256sum`
- `python3`
- `docker`
- Docker Compose plugin
- `pg_dump` 通过 postgres 容器执行，不要求宿主机安装 PostgreSQL client

服务器禁止执行：

- `npm run build`
- `mvn package`
- `docker build`
- `docker pull`
- 从源码仓库拉代码后现场构建

### 5.3 Jenkins

第一版不引入 Jenkins。未来如果需要 Web UI、审批、定时构建或团队流水线，可以引入 Jenkins，但 Jenkins 只能调用本仓库脚本，不在 Jenkins Job 内编写核心逻辑。

## 6. 版本与源码规则

### 6.1 发布版本

发布版本必须匹配：

```text
^V[0-9]+\.[0-9]+\.[0-9]+$
```

示例：

```text
V1.0.0
V1.0.1
V1.1.0
```

### 6.2 Flyway 版本强绑定

发布版本必须等于 ArtFetch 最新 Flyway 迁移版本。最新迁移从以下目录读取：

```text
/Users/wyn/code/ArtFetch/backend/src/main/resources/db/migration
```

如果本次发布没有真实 schema 变更，也必须在 ArtFetch 中新增对应版本的 Flyway marker 迁移文件，确保发布版本和最新 Flyway 版本相等。

### 6.3 源码洁净度

默认只允许从干净 Git 工作区发布：

- ArtFetch 必须是 Git 仓库。
- 默认要求 `git status --porcelain` 为空。
- manifest 记录 ArtFetch 分支、commit、短 SHA 和 dirty 状态。

`--allow-dirty` 规则：

- 必须显式传入。
- 只允许标记为临时或应急发布。
- manifest 记录 `sourceDirty: true`。
- 构建报告记录 `git diff --stat` 和 `git diff --name-only`。
- 不得标记为正式生产发布。

## 7. 发布包设计

### 7.1 本地制品库

本地发布制品保存到：

```text
artifacts/releases/<version>/
├── artfetch-deploy-<version>.tgz
├── artfetch-deploy-<version>.tgz.sha256
├── release-manifest.json
└── build-report.md
```

默认保留最近 5 个成功发布版本，可通过环境变量调整：

```text
ARTFETCH_RELEASE_RETENTION=5
```

### 7.2 发布包内部结构

发布包必须是完整可部署单元：

```text
artfetch-deploy-<version>/
├── docker-compose.prod.yml
├── .env.example
├── release-manifest.json
├── images/
│   ├── artfetch-backend-<version>.tar.gz
│   ├── artfetch-backend-<version>.tar.gz.sha256
│   ├── artfetch-frontend-<version>.tar.gz
│   ├── artfetch-frontend-<version>.tar.gz.sha256
│   ├── artfetch-jupyter-<version>.tar.gz
│   ├── artfetch-jupyter-<version>.tar.gz.sha256
│   ├── postgres-16-alpine.tar.gz
│   └── postgres-16-alpine.tar.gz.sha256
└── scripts/
    ├── install-or-upgrade.sh
    ├── verify-deployment.sh
    ├── backup-db.sh
    ├── rollback.sh
    └── clean-failed-install.sh
```

发布包禁止包含：

- `.env`
- 数据库 dump
- 生产密码
- 雅昌 Cookie、账号或密码
- 对象存储 Access Key 或 Secret Key
- SSH 私钥
- GitHub token

### 7.3 运行镜像范围

每个发布包包含全部运行所需镜像：

- `artfetch-backend:<version>`
- `artfetch-frontend:<version>`
- `artfetch-jupyter:<version>`
- `postgres:16-alpine`

自有服务镜像同时打版本 tag 和源码 SHA tag：

```text
artfetch-backend:<version>
artfetch-backend:sha-<sourceGitSha>
artfetch-frontend:<version>
artfetch-frontend:sha-<sourceGitSha>
artfetch-jupyter:<version>
artfetch-jupyter:sha-<sourceGitSha>
```

基础镜像只在本地构建时使用，不进入发布包，除非它本身也是 Docker Compose 运行时直接引用的镜像。

### 7.4 Manifest

`release-manifest.json` 是发布包、部署、回滚和审计的事实来源。

必须记录：

- app 名称。
- release version。
- Flyway version。
- ArtFetch source repo。
- source branch。
- source Git SHA。
- source dirty 状态。
- builtAt。
- build host。
- build user。
- 每个镜像的 version tag、sha tag、image id、tar 路径和 tar SHA256。
- compose 文件路径和 SHA256。
- 构建结果：
  - frontend build。
  - backend package。
  - backend test。
  - image build。
  - image export。

校验规则：

- `app` 必须等于 `artfetch`。
- `version` 必须匹配 `Vx.y.z`。
- `version` 必须等于 `flywayVersion`。
- 每个镜像 tar 必须存在且 SHA256 匹配。
- Compose 文件 SHA256 必须匹配。
- manifest 不得包含任何密钥值。

## 8. 脚本与命令

脚本是第一版唯一自动化事实来源。所有脚本必须支持 `--help`，失败时返回非零退出码。

### 8.1 标准命令

预检：

```bash
scripts/local/preflight.sh --version V1.0.2 --source /Users/wyn/code/ArtFetch
```

构建离线发布包：

```bash
scripts/release/build-release.sh --version V1.0.2 --source /Users/wyn/code/ArtFetch
```

部署指定版本：

```bash
scripts/deploy/deploy-release.sh --version V1.0.2 --env prod
```

组合入口：

```bash
scripts/release-and-deploy.sh --version V1.0.2 --env prod
```

### 8.2 建议目录

```text
ArtFetchDeploy/
├── README.md
├── docs/
├── config/
│   └── environments/
│       ├── prod.env
│       └── staging.env.example
├── templates/
│   ├── docker-compose.prod.yml
│   └── .env.example
├── scripts/
│   ├── local/
│   │   └── preflight.sh
│   ├── release/
│   │   ├── build-release.sh
│   │   └── verify-release-package.sh
│   ├── deploy/
│   │   ├── deploy-release.sh
│   │   └── rollback-release.sh
│   ├── remote/
│   │   ├── install-or-upgrade.sh
│   │   ├── verify-deployment.sh
│   │   ├── backup-db.sh
│   │   ├── rollback.sh
│   │   └── clean-failed-install.sh
│   └── release-and-deploy.sh
├── artifacts/
├── reports/
└── logs/
```

`scripts/remote/` 中的脚本由 build-release 打进发布包。服务器执行的是发布包内脚本，不依赖服务器提前存在这些脚本。

## 9. 功能需求

### 9.1 本地预检

检查项：

- ArtFetch 源码目录存在。
- ArtFetch 是 Git 仓库。
- ArtFetch 工作区默认干净。
- 发布版本格式合法。
- 发布版本等于最新 Flyway 迁移版本。
- 必需 Dockerfile 存在：
  - `backend/Dockerfile`
  - `frontend/Dockerfile`
  - `ml/Dockerfile`
- 本地 Docker 可用。
- 本地 Docker Compose plugin 可用。
- Node.js、npm、Java、Maven、Python 3 可用。
- `templates/docker-compose.prod.yml` 存在且可通过后续变量渲染。
- 目标环境 profile 存在。

验收标准：

- 任一硬门禁失败时退出非零。
- 输出失败原因和修复建议。
- 不输出密钥值。

### 9.2 本地构建

第一版强制全量构建：

- `frontend`: `npm ci && npm run build`
- `backend`: `mvn package -DskipTests`
- `jupyter`: 使用 `ml/Dockerfile` 构建镜像。
- Docker build：
  - backend
  - frontend
  - jupyter
- 拉取本地构建所需的运行外部镜像：
  - `postgres:16-alpine`

后端测试：

- 第一版不作为默认硬门禁。
- `--with-tests` 时执行后端测试。
- manifest 记录 `backendTest: skipped|passed|failed`。

验收标准：

- 前端 build 失败不得发布。
- 后端 package 失败不得发布。
- 任一 Docker image build 失败不得发布。
- 任一 image export 或 SHA256 校验失败不得发布。

### 9.3 发布包生成与校验

需求：

- 生成发布包目录。
- 复制 ArtFetchDeploy 维护的生产 Compose 和 `.env.example`。
- 打入服务器脚本。
- 导出所有运行镜像 tar.gz。
- 为每个 tar.gz 生成 SHA256。
- 生成 manifest。
- 打包 `artfetch-deploy-<version>.tgz`。
- 生成顶层 `.tgz.sha256`。
- 重新解包并校验发布包。

验收标准：

- 发布包可在没有 ArtFetch 源码的机器上完成部署。
- manifest 与实际文件完全一致。
- 发布包不包含密钥。

### 9.4 环境 Profile

第一版支持环境 profile，先落 `prod`，预留 `staging`。

配置文件示例：

```text
config/environments/prod.env
config/environments/staging.env.example
```

非敏感配置示例：

```env
ARTFETCH_SSH_TARGET=artfetch-prod
ARTFETCH_PROJECT_DIR=/opt/artfetch
ARTFETCH_BASE_URL=http://127.0.0.1:3000
ARTFETCH_PUBLIC_URL=http://124.174.79.81:3000
ARTFETCH_REMOTE_USE_SUDO=0
ARTFETCH_RELEASE_RETENTION=5
```

禁止写入环境 profile：

- SSH 私钥。
- SSH 密码。
- PostgreSQL 密码。
- ArtFetch 管理员密码。
- 雅昌 Cookie。
- 对象存储密钥。
- GitHub token。

### 9.5 服务器首次安装

本地部署脚本负责：

- 校验本地发布包存在。
- 上传发布包和 SHA256 到服务器临时目录。
- 在服务器校验包 SHA256。
- 解包到服务器 release 目录。
- 调用包内 `scripts/install-or-upgrade.sh`。

服务器脚本负责：

- 检查必需命令。
- 创建项目目录。
- 创建 `releases/`、`backups/`、`backend/logs/`、`storage/original-images/`。
- 如果 `.env` 不存在，从包内 `.env.example` 创建。
- 自动生成强随机值：
  - `POSTGRES_PASSWORD`
  - `ARTFETCH_ADMIN_PASSWORD`
  - `ARTFETCH_OBJECT_STORAGE_ENCRYPTION_KEY`
- 生成 `.env.release`。
- `docker load` 所有镜像。
- `docker compose up -d`。
- 执行健康检查。

验收标准：

- `.env` 和 `.env.release` 权限为 `600`。
- 日志不打印密钥。
- 首次安装后容器和 HTTP 健康检查通过。

### 9.6 后续升级

升级前硬门禁：

- 当前服务器 Docker 可用。
- 当前项目目录可读写。
- 发布包 SHA256 校验通过。
- manifest 校验通过。
- 运行任务检查通过。
- 数据库备份成功且文件非空。

运行任务检查：

- `search_tasks.status = 'RUNNING'`
- `hd_image_migration_tasks.status = 'RUNNING'`
- `hd_image_migration_items.status = 'UPLOADING'`

发现运行任务时默认停止部署。只有显式 `--force` 才允许继续，并且部署报告必须标记风险。

升级步骤：

- 上传并校验发布包。
- 解包到 `/opt/artfetch/releases/<version>/`。
- 执行升级前数据库备份。
- `docker load` 新版本镜像。
- 写入新的 `.env.release`。
- 替换 `docker-compose.prod.yml`。
- 启动服务。
- 执行健康检查。
- 写入部署历史。

### 9.7 数据库备份

已有部署升级前必须备份 PostgreSQL：

```text
/opt/artfetch/backups/db-before-<target-version>-<timestamp>.dump
```

要求：

- 使用 `pg_dump -Fc`。
- 通过 postgres 容器执行。
- 备份文件必须非空。
- 备份失败停止部署。
- 部署报告记录备份路径和大小。

首次安装且没有既有数据库时可以不备份。

### 9.8 健康检查

第一版只做容器和 HTTP 健康检查，不做登录级 smoke test。

自动检查项：

- Docker Compose 配置可解析。
- 容器存在且状态稳定。
- Postgres healthy。
- Backend `/actuator/health` 返回 UP。
- Frontend `/` 返回 `2xx` 或 `3xx`。
- `/api/auth/me` 未登录返回 `401`。
- 服务器当前 `release-manifest.json` 版本等于目标版本。

第一版不做：

- 自动登录。
- 管理员账号验证。
- 创建测试任务。
- Excel 导出验证。
- 人工 smoke test 强制门槛。

### 9.9 回滚

默认回滚范围：

- 选择上一版成功部署包。
- `docker load` 上一版镜像。
- 恢复上一版 `docker-compose.prod.yml`。
- 恢复上一版 `.env.release`。
- `docker compose up -d`。
- 执行健康检查。

数据库不自动恢复。

如果目标回滚版本的 Flyway 版本小于当前数据库已执行版本，默认阻止应用回滚，并提示需要数据库恢复计划。

数据库恢复必须显式传入类似参数：

```text
--restore-db /opt/artfetch/backups/<dump-file> --i-understand-data-loss-risk
```

恢复数据库前必须再次备份当前数据库。

### 9.10 失败清理

安装或升级失败时默认不删除持久化数据。

允许清理：

- 本次失败产生的临时解包目录。
- 候选 `.env.release`。
- 候选 Compose 文件。
- 未成功启用的应用容器。

必须保留：

- `.env`
- PostgreSQL 数据卷。
- 图片目录。
- 后端日志。
- 数据库备份。
- 部署历史。

完全重装并删除数据必须显式设置：

```text
ARTFETCH_WIPE_DATA=1
```

没有该变量时，脚本不得删除持久化数据。

### 9.11 制品保留

本地和服务器默认保留最近 5 个成功版本。

清理规则：

- 只删除旧发布包和旧解包目录。
- 不删除当前运行版本。
- 不删除数据库备份。
- 不删除 `.env`。
- 不删除部署历史。
- 清理前输出将删除的版本列表。

## 10. 安全要求

- 脚本默认使用严格模式。
- 不打印 `.env` 中的真实值。
- 不把 token、密码或 Cookie 写入日志、报告或 manifest。
- 生产 `.env` 只存在于服务器。
- `.env` 和 `.env.release` 权限为 `600`。
- 环境 profile 只保存非敏感配置。
- 默认要求远程用户可直接运行 Docker。
- 如需 sudo，必须配置 `ARTFETCH_REMOTE_USE_SUDO=1`，且要求免密 sudo。
- PostgreSQL、后端调试端口和 Jupyter 默认不暴露公网。

## 11. 报告与审计

构建报告记录：

- 版本。
- ArtFetch 分支和 Git SHA。
- dirty 状态。
- Flyway 版本。
- 构建命令结果。
- 镜像 tag、image id 和 tar SHA256。
- 发布包路径和 SHA256。

部署报告记录：

- 版本。
- 环境 profile。
- SSH target。
- 服务器项目目录。
- 上传包路径。
- 数据库备份路径。
- 运行任务检查结果。
- 健康检查结果。
- 开始和结束时间。
- 执行人。
- `--force` 或 `--allow-dirty` 风险标记。

报告不得包含密钥值。

## 12. 里程碑

### M1：需求与项目骨架

- 完成需求文档。
- 完成 README。
- 完成 `.gitignore`。
- 完成目录结构。
- 完成环境 profile 模板。
- 完成生产 Compose 和 `.env.example` 模板。

验收：可以明确知道 ArtFetchDeploy 是本地脚本事实来源，不依赖 ArtFetch 旧 workflow 或部署脚本。

### M2：预检与本地构建

- 实现 `scripts/local/preflight.sh`。
- 实现 `scripts/release/build-release.sh`。
- 实现 `scripts/release/verify-release-package.sh`。
- 生成完整本地离线发布包。

验收：不连接生产服务器也能判断一个版本是否可发布。

### M3：部署与健康检查

- 实现环境 profile 加载。
- 实现 `scripts/deploy/deploy-release.sh`。
- 实现包内 `install-or-upgrade.sh`。
- 实现包内 `backup-db.sh`。
- 实现包内 `verify-deployment.sh`。

验收：能把本地发布包上传到 `prod` 并完成健康检查。

### M4：回滚、清理与审计

- 实现应用回滚。
- 实现失败清理。
- 实现制品保留策略。
- 实现部署历史和报告。

验收：一次失败部署可以被定位、清理、重试或回滚。

### M5：稳定化

- 增加 dry-run。
- 增加 `--with-tests`。
- 增加 `staging` 实际环境。
- 评估是否引入 Jenkins 作为脚本调用外壳。

## 13. 验收总标准

本项目达到第一版可用状态时，应满足：

- 不依赖 ArtFetch 旧 GitHub Actions workflow。
- 不依赖 ArtFetch 旧发布或部署脚本。
- 可以从本地对 ArtFetch 源码执行预检。
- 可以本地全量构建完整离线发布包。
- 可以校验发布包完整性。
- 可以通过 SSH/SCP 上传并部署到生产服务器。
- 服务器不构建源码、不拉 registry 镜像。
- 升级前强制备份数据库。
- 升级前检查运行任务。
- 部署后完成容器和 HTTP 健康检查。
- 失败时不删除持久化数据。
- 可以默认应用回滚，数据库恢复需人工显式确认。
- 构建、部署和回滚报告不包含密钥。

## 14. 风险与对策

| 风险 | 影响 | 对策 |
| --- | --- | --- |
| 本地构建机环境不一致 | 构建结果不可重复 | 预检记录工具版本，后续可容器化构建环境 |
| 发布版本和 Flyway 版本不一致 | 数据库迁移不可控 | 发布前硬门禁，必须相等 |
| 使用 dirty 源码发布 | 难以追溯 | 默认禁止，显式 `--allow-dirty` 并记录风险 |
| 服务器误删数据 | 数据丢失 | 默认禁止删除持久化数据，wipe 必须显式开启 |
| 升级打断运行任务 | 业务异常 | 运行任务检查作为硬门禁，`--force` 记录风险 |
| 数据库回滚误操作 | 数据丢失 | 默认不恢复数据库，恢复需显式高风险确认 |
| 脚本和 Jenkins 配置漂移 | 自动化不可审计 | 脚本是事实来源，Jenkins 只做调用外壳 |
| ArtFetch Dockerfile 不适合生产 | 构建或运行不稳定 | 第一版复用，后续可迁移生产 Dockerfile 到 ArtFetchDeploy |

