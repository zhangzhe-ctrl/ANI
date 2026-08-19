---
kind: build_system
name: ANI Monorepo 构建与发布流水线（Makefile + GitHub Actions + Helm/Compose）
category: build_system
scope:
    - '**'
source_files:
    - repo/Makefile
    - repo/go.work
    - .github/workflows/ci.yml
    - repo/.github/workflows/build-image.yml
    - repo/deploy/docker/docker-compose.yml
    - repo/deploy/helm/ani-platform/Chart.yaml
    - api/proto/buf.yaml
    - api/proto/buf.gen.yaml
    - scripts/list_go_modules.py
    - ci/requirements-contract.txt
---

## 1. 使用的系统与方法

- **统一入口**：仓库根目录 `repo/Makefile` 是 monorepo 的单一编排入口，集中定义代码生成、Go/Python/前端构建、测试、校验门禁、本地依赖启动等全部目标。
- **Go 多模块工作区**：通过 `repo/go.work` 声明 `cli/ani`、`pkg`、`services/*`、`tools/kms-sm4-live-fixture` 等子模块为同一 workspace，所有 Go 构建/测试都基于此工作区执行。
- **CI 流水线**：`.github/workflows/ci.yml` 在 PR/Push 上并行运行 Go 构建+测试、Python AI 层 lint/test、前端 build/type-check/lint、Services 边界/契约门禁、OpenAPI spec lint、依赖 CVE 扫描，并以 `required-gates` job 强制所有 gate 通过。
- **镜像构建与发布**：`repo/.github/workflows/build-image.yml` 在 `v*.*.*` tag push 或手动触发时，使用 Docker Buildx 以 `linux/amd64,linux/arm64` 双平台构建并推送至 Harbor (`harbor.ani.internal/ani`)，随后执行 Trivy 安全扫描与 Cosign 签名。
- **部署产物**：Helm Chart `repo/deploy/helm/ani-platform/Chart.yaml` 作为 umbrella chart；Kubernetes 清单按 profile 分目录存放于 `deploy/manifests/m1-*`；本地开发依赖通过 `deploy/docker/docker-compose.yml` 管理。

## 2. 关键文件

- `repo/Makefile` — 统一构建/测试/校验门禁入口（约 1000 行，覆盖所有服务与 sprint gate）
- `repo/go.work` / `repo/go.work.sum` — Go workspace 声明与工作区缓存
- `.github/workflows/ci.yml` — PR 主 CI 流水线（Go/Python/Frontend/Services Gate/OpenAPI）
- `repo/.github/workflows/build-image.yml` — 镜像构建、推送、Trivy 扫描、Cosign 签名
- `repo/deploy/docker/docker-compose.yml` — 本地 PG/MinIO/NATS/Redis/Milvus 依赖栈
- `repo/deploy/helm/ani-platform/Chart.yaml` — Helm umbrella chart 元数据
- `api/proto/buf.yaml` / `api/proto/buf.gen.yaml` — Protobuf 代码生成配置
- `scripts/list_go_modules.py` — 被 CI 和 Makefile 共同用于枚举 Go 模块
- `ci/requirements-contract.txt` — Services 契约校验 Python 依赖
- `ruff.toml` — Python 代码风格规则

## 3. 架构与约定

### 构建流程
- Go 二进制通过 `make build` 聚合 `build-gateway`、`build-auth-service`、`build-model-service`、`build-task-service`、`build-reconcile-worker`、`build-cli` 六个 target，每个 target 在各自 `services/*/` 目录下执行 `go build -ldflags "$(LDFLAGS)"`，输出到 `repo/bin/`。版本信息通过 `-X main.Version` 注入，值来自 `git describe --tags --always --dirty`。
- 代码生成链：`make gen-api` → services/ani-gateway 的 `go generate` 从 OpenAPI 生成 Gateway Go 代码，同时用 `openapi-typescript` 生成 Console TypeScript 类型；`make gen-proto` 调用 buf 从 `api/proto` 生成 gRPC Go 代码；`make gen-core-sdk` 调用 Python 脚本生成四语言 SDK Alpha。
- 测试分层：`make test-go` 在工作区范围跑 `./pkg/... ./services/...`；`make test-python` 仅对 RAG Engine 做语法编译检查；覆盖率由 `make test-cover` 生成 `coverage.html`。
- 校验门禁（validate-*）：大量 `make validate-*` target 对应各 Sprint/特性，调用 `scripts/validate_*.py` 或 go test 特定函数，形成“契约即门禁”的验证体系。

### CI 策略
- `go-ci`：安装 golangci-lint、gosec，逐模块运行 `go test -coverprofile`，并通过 `python scripts/list_go_modules.py` 发现所有 Go module。
- `python-ci`：安装 `ai/rag-engine/requirements.txt`，执行 ruff/mypy/pytest，并用 `validate_python_test_policy.py` 强制 Python AI 层测试策略。
- `frontend-ci`：Node 20，npm ci → audit → type-check → lint → build。
- `services-pr-gate`：安装 `ci/requirements-contract.txt`，执行 `make validate-services`、`make validate-ci-workflow`、`make validate-doc-entrypoints`。
- `required-gates`：汇总所有 gate 结果，任一失败则 PR 被阻断。

### 镜像与发布
- 触发条件：tag `v*.*.*` push 或 workflow_dispatch 传入 tag。
- 矩阵构建：`ani-gateway`、`inference-operator`、`upgrade-operator`、`rag-engine`、`doc-parser` 五个服务分别以各自 context 构建，输出到 `harbor.ani.internal/ani/<service>:<tag>` 及 `:latest`。
- 缓存：使用 registry-backed build cache (`type=registry,ref=...:buildcache`)。
- 安全：Trivy 阻断 HIGH/CRITICAL；Cosign 使用 `COSIGN_PRIVATE_KEY` 签名镜像。

### 本地开发
- `make deps` 启动 docker-compose 中的 PostgreSQL、MinIO、NATS、Redis、Milvus，并等待 PG 就绪。
- `make dev-gateway`、`make dev-console`、`make dev-core-mock` 提供热重载开发入口。
- `.env` 自动加载并 export 给子进程，便于本地环境变量注入。

## 4. 约定与约束

- **Go 构建必须 CGO_ENABLED=0**：所有 `build-*` target 显式设置 `CGO_ENABLED=0`，确保静态链接可移植二进制。
- **版本注入来源固定**：`VERSION` 默认取 `git describe --tags --always --dirty`，未打 tag 时为 `dev`。
- **Go 模块枚举权威源**：CI 与 Makefile 均通过 `python scripts/list_go_modules.py` 动态发现模块，避免硬编码路径。
- **PR 必须通过全部 required gates**：`required-gates` job 显式检查 `go-ci`、`python-ci`、`frontend-ci`、`services-pr-gate`、`api-spec-lint` 均为 success，否则合并被阻断。
- **Python AI 层测试策略受控**：CI 中 `validate_python_test_policy.py` 强制执行 Python 测试策略，新增测试需满足策略要求。
- **镜像安全门槛**：Trivy 以 `severity: HIGH,CRITICAL` 阻断发布；Cosign 签名密钥通过 secrets 注入。
- **Helm Chart 声明依赖**：`Chart.yaml` 通过注解 `ani.kubercloud.io/dependencies` 声明平台依赖（postgresql、nats、redis、minio、milvus、harbor），作为部署契约的一部分。
- **Protobuf 工具版本锁定**：`make tools-proto` 固定安装 `protoc-gen-go@v1.33.0`、`protoc-gen-go-grpc@v1.3.0`、`grpc-gateway v2.19.1`，保证跨环境一致。
- **OpenAPI 是唯一真实来源**：Gateway Go 代码、Console TS 类型、SDK 均由 `api/openapi/services/v1.yaml` 与 Core OpenAPI 生成，禁止手写接口定义。