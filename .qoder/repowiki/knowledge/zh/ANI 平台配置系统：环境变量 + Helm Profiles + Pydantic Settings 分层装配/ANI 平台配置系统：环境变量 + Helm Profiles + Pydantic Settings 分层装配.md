---
kind: configuration_system
name: ANI 平台配置系统：环境变量 + Helm Profiles + Pydantic Settings 分层装配
category: configuration_system
scope:
    - '**'
source_files:
    - repo/.env.example
    - repo/pkg/bootstrap/server.go
    - repo/pkg/bootstrap/deps.go
    - repo/services/auth-service/internal/config/config.go
    - repo/services/model-service/internal/config/config.go
    - repo/services/task-service/internal/config/config.go
    - repo/services/reconcile-worker/internal/config/config.go
    - repo/services/kb-service/app/core/config.py
    - repo/ai/rag-engine/app/core/config.py
    - repo/deploy/helm/ani-platform/values.yaml
    - repo/deploy/helm/ani-platform/profiles/dev.yaml
---

## 1. 总体方案

ANI 平台采用 **环境变量优先、Helm Profile 覆盖、Pydantic Settings 校验** 的分层配置模型，Go 服务与 Python 服务各自实现但遵循同一约定：所有运行时参数通过环境变量注入，部署时由 Helm Chart 的 `values.yaml` + `profiles/*.yaml` 生成 Kubernetes Secret/ConfigMap/环境变量，开发环境使用根目录 `.env.example`（复制为 `.env`）作为共享源。

- Go 微服务统一通过 `pkg/bootstrap.Config` 描述连接串与能力开关，再由各服务的 `internal/config/config.go` 从 `os.Getenv` 读取并填充；启动时调用 `bootstrap.MustConnect(cfg)` 一次性建立 Postgres/NATS/Redis 连接并组装 `Capabilities` 适配器。`server.go` 中的 `withEnvironmentOverrides()` 在进程启动时把一组 `WORKLOAD_*` / `NETWORK_*` / `STORAGE_*` / `OBJECT_STORE_*` / `VECTOR_STORE_*` / `KUBERNETES_*` / `REDIS_*` 等环境变量直接覆写 `Config` 字段，形成“代码默认值 → 环境变量”的单一覆盖点。
- Python 服务（kb-service、ai/rag-engine）使用 `pydantic_settings.BaseSettings`，通过 `SettingsConfigDict(env_file=".env", extra="ignore")` 加载共享 `.env`，并用 `Field(validation_alias=...)` 同时接受新旧变量名（如 `DATABASE_URL` / `PG_DSN`、`ANI_GATEWAY_INTERNAL_URL` / `ANI_GATEWAY_URL`），保证向后兼容。
- Helm Chart (`repo/deploy/helm/ani-platform`) 以 `values.yaml` 为基线，`profiles/` 下按部署场景拆分（dev、attach-k8s、offline、gpu-scheduling、instance-foundation、runtime-foundation、cluster-validation），通过 `global.profile` 选择基础设施 profile，并以 `infrastructure.profiles.*.mode: external|bundled` 控制是否安装依赖。

## 2. 关键文件与包

- Go 配置核心：
  - `repo/pkg/bootstrap/server.go` — `Config` 结构体定义全部可配置项（数据库、NATS、Redis、对象存储、向量库、工作负载/网络/存储 provider、Kubernetes 客户端、reconcile 控制器开关等），以及 `MustConnect`、`RunGRPC`、`RunHealthProbe`、`withEnvironmentOverrides()`。
  - `repo/pkg/bootstrap/deps.go` — `NewCapabilitiesWithConfig` 根据 `Config` 中的 provider 字符串（`local` / `kubernetes_rest` / `kubeovn_rest` / `minio` / `milvus` / `harbor` / `prometheus_kubernetes` 等）动态拼装适配器，未配置时返回 `NotConfigured` 空实现。
  - 各服务 `internal/config/config.go`：auth-service、model-service、task-service、metering-service、reconcile-worker，均复用 `bootstrap.Config` 并追加领域字段（如 `OutboxEnabled`、OIDC 密钥、JWT issuer）。
- Python 配置：
  - `repo/services/kb-service/app/core/config.py` — kb-service 的 `Settings`，映射 DATABASE_URL / NATS_URL / REDIS_URL / ANI_GATEWAY_INTERNAL_URL。
  - `repo/ai/rag-engine/app/core/config.py` — rag-engine 的 `Settings`，含 Milvus、Embedding API、VLLM、MinIO、OCR 等字段，并通过 `field_validator` 将 `MILVUS_ADDR` 拆成 host/port。
- 环境变量模板与契约：
  - `repo/.env.example` — 全平台共享的环境变量清单（数据库、MinIO、NATS、Redis、Milvus、JWT/OIDC、Gateway 端口、RAG VLLM 等），明确标注不得提交到版本控制。
- Helm 部署配置：
  - `repo/deploy/helm/ani-platform/values.yaml` — 顶层 `global.profile`、`infrastructure.profiles`（dev/attach-k8s/offline/gpuScheduling/gpuSchedulingE2E/runtimeFoundation/instanceFoundation）、各组件 `mode: external|bundled`、GPU/实例/网络/存储 provider 默认值。
  - `repo/deploy/helm/ani-platform/profiles/*.yaml` — 场景化覆盖文件，例如 `dev.yaml` 将所有基础设施设为 `external` 并指向本地镜像仓库。

## 3. 架构与约定

### 3.1 配置来源优先级
1. **代码默认值**：各 `config.Load()` / `BaseSettings` 中定义的默认值（通常指向 `localhost` 开发地址）。
2. **`.env` 文件**：Python 服务通过 pydantic-settings 自动加载；Go 服务不直接读 `.env`，但 `.env.example` 是跨服务约定的变量命名来源。
3. **进程环境变量**：`bootstrap.server.go` 的 `withEnvironmentOverrides()` 在启动时扫描大量 `WORKLOAD_*` / `NETWORK_*` / `STORAGE_*` / `OBJECT_STORE_*` / `VECTOR_STORE_*` / `KUBERNETES_*` / `REDIS_*` 环境变量并覆写 `Config`；Go 服务自己的 `config.Load()` 也通过 `os.Getenv` 读取服务级变量（如 `AUTH_OIDC_*`、`OUTBOX_*`）。
4. **Helm values/profiles**：通过 `helm install/upgrade --values values.yaml --values profiles/dev.yaml ...` 叠加，最终由 Helm template 渲染为 Pod 的 envFrom/Secret 引用。

### 3.2 Provider 模式（Feature Flag 式配置）
几乎所有外部能力都以 `<X>_PROVIDER` 环境变量切换实现：
- `WORKLOAD_PROVIDER` = `local` | `kubernetes_rest`
- `NETWORK_PROVIDER` = `local` | `kubeovn_rest`
- `STORAGE_PROVIDER` = `local` | `kubernetes_rest`
- `OBJECT_STORE_PROVIDER` = `minio` | `not_configured`
- `VECTOR_STORE_PROVIDER` = `milvus` | `not_configured`
- `REGISTRY_PROVIDER_MODE` = `harbor` | `not_configured`
- `INSTANCE_OBSERVABILITY_PROVIDER` = `prometheus_kubernetes` | `not_configured`
- `GPU_INVENTORY_PROVIDER` = `kubernetes_rest` | `not_configured`
- `WORKLOAD_LIFECYCLE_PROVIDER` / `WORKLOAD_OPS_PROVIDER`

当值为 `local` / `not_configured` / 空字符串时，`deps.go` 返回对应 `NotConfigured` 或本地模拟实现，使服务可在无 K8s/MinIO/Milvus 的环境下启动（仅功能降级）。这是平台支持 dev/local/offline/gpu-scheduling-e2e 等多种 profile 的核心机制。

### 3.3 健康检查与探针
`bootstrap.RunGRPC` 会额外启动一个独立的 HTTP 服务器监听 `HEALTH_PORT`，提供依赖健康探针（DB/NATS/Redis/K8s 连通性），供 K8s liveness/readiness 探测使用。`RunHealthProbe` 则用于不需要 gRPC 的服务（如 metering-service）。

### 3.4 配置验证与错误处理
- Go：`bootstrap.MustConnect` 在 DB/NATS/Redis 任一连接失败时 `os.Exit(1)`，provider 不支持时返回 `ports.ErrUnsupported` 错误；`parseBool` / `parseInt` 对非法值回退到默认值而非抛错，保证健壮性。
- Python：`pydantic_settings` 在启动时校验类型，`extra="ignore"` 允许其他服务的环境变量被安全忽略，避免跨服务污染。

## 4. 约定与约束

- **环境变量命名规范**：所有服务共享 `DATABASE_URL` / `NATS_URL` / `REDIS_URL` / `MINIO_*` / `MILVUS_ADDR` / `AUTH_*` / `GATEWAY_*` / `VLLM_*` 等变量名（见 `.env.example`），新增配置应沿用此命名风格。
- **`.env` 禁止入仓**：`.env.example` 顶部注释明确 `.env` 不得提交到版本控制（已在 `.gitignore` 中排除）。
- **Provider 白名单**：`deps.go` 中每个 switch 仅接受显式列出的 provider 字符串，未知值返回 `ErrUnsupported`，防止拼写错误导致静默回退。
- **Profile 驱动部署**：生产/离线/GPU/e2e 等场景通过 `--values profiles/<name>.yaml` 切换，不在代码中硬编码环境判断；`global.profile` 字段贯穿 values 树，便于模板引用。
- **Kubernetes Secret 承载敏感信息**：Helm values 中 `secretName: ani-bootstrap-placeholders` + `secretKey: database-url/nats-url/redis-url` 表明数据库、NATS、Redis 连接串通过 Kubernetes Secret 注入，而非明文写入 values。
- **向后兼容别名**：Python 配置使用 `validation_alias` 同时支持新旧变量名（如 `pg_dsn`/`database_url`/`DATABASE_URL`/`PG_DSN`、`ani_gateway_url`/`ani_gateway_internal_url`/`ANI_GATEWAY_URL`/`ANI_GATEWAY_INTERNAL_URL`），迁移期不破坏既有部署。
- **健康端口隔离**：gRPC 服务与健康探针 HTTP 服务分离（`GRPC_PORT` vs `HEALTH_PORT`），gateway 单独通过 `GATEWAY_LISTEN_ADDR` 控制监听地址，避免端口冲突。
