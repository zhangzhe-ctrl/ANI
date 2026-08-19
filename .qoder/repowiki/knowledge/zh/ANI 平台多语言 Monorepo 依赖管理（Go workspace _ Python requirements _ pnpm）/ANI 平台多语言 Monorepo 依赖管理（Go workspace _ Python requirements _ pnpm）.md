---
kind: dependency_management
name: ANI 平台多语言 Monorepo 依赖管理（Go workspace / Python requirements / pnpm）
category: dependency_management
scope:
    - '**'
source_files:
    - repo/go.work
    - repo/go.work.sum
    - repo/pkg/go.mod
    - repo/cli/ani/go.mod
    - repo/services/ani-gateway/go.mod
    - repo/tools/kms-sm4-live-fixture/go.mod
    - repo/ai/rag-engine/requirements.txt
    - repo/ci/requirements-contract.txt
    - repo/frontends/console/package.json
    - repo/frontends/console/pnpm-lock.yaml
    - repo/Makefile
    - repo/.github/workflows/build-image.yml
---

## 1. 使用的系统与工具

本仓库是一个多语言 Monorepo，按语言分别采用不同的依赖管理系统：

- **Go**：使用 Go 官方 `go work` 工作区（workspace），根目录 `repo/go.work` 声明所有子模块；每个服务/工具拥有独立的 `go.mod` + `go.sum`，通过 `replace` 指向本地共享包。
- **Python**：RAG 引擎 `ai/rag-engine/requirements.txt` 固定第三方库版本；CI 校验脚本的依赖集中在 `ci/requirements-contract.txt`。
- **前端（Console）**：基于 `pnpm` 工作区（`frontends/console/pnpm-workspace.yaml`、`pnpm-lock.yaml`），`package.json` 声明依赖与 `overrides`。
- **构建/代码生成**：统一入口为根 `Makefile`，封装 `go build`、`buf generate`、`openapi-typescript`、`protoc-gen-go` 等工具安装与调用。
- **镜像构建与发布**：GitHub Actions `.github/workflows/build-image.yml` 通过 Docker Buildx 构建并推送至私有 Harbor 仓库 `harbor.ani.internal/ani`，并使用 Trivy 扫描漏洞、Cosign 签名镜像。

## 2. 关键文件

| 类别 | 文件 | 作用 |
|---|---|---|
| Go 工作区 | `repo/go.work` | 声明 Go 1.25.0 及 9 个被 use 的模块路径 |
| Go 工作区锁 | `repo/go.work.sum` | 锁定工作区内所有间接依赖的哈希 |
| 共享包 | `repo/pkg/go.mod` | 定义 `github.com/kubercloud/ani/pkg` 及其 PG/NATS/Redis/gRPC 依赖 |
| CLI 模块 | `repo/cli/ani/go.mod` | 仅声明 module 与 go 版本，依赖由 workspace 解析 |
| 服务模块 | `repo/services/ani-gateway/go.mod` | 引用 `../../pkg` via `replace`，引入 Hertz、gRPC 等 |
| 工具模块 | `repo/tools/kms-sm4-live-fixture/go.mod` | 同样 `replace` 指向本地 pkg |
| Python 依赖 | `repo/ai/rag-engine/requirements.txt` | 固定 FastAPI、LlamaIndex、Milvus、NATS 等版本 |
| CI 校验依赖 | `repo/ci/requirements-contract.txt` | PyYAML、openapi-spec-validator、pytest 固定版本 |
| 前端依赖 | `repo/frontends/console/package.json` + `pnpm-lock.yaml` | React/TanStack/TDesign 依赖及 lockfile |
| 构建入口 | `repo/Makefile` | 封装 deps、gen-api、build、test、validate-* 等全部命令 |
| CI 流水线 | `repo/.github/workflows/build-image.yml` | 构建/推送/扫描/签名镜像到 Harbor |

## 3. 架构与约定

### 3.1 Go 依赖：Workspace + replace
- 所有 Go 模块（CLI、pkg、各 service、tools fixture）均声明 `go 1.25.0`，并通过 `go.work` 聚合。新增服务需手动在 `go.work` 的 `use (...)` 块中添加路径。
- 跨模块引用统一通过 `replace github.com/kubercloud/ani/pkg => ../../pkg` 实现，避免发布到远端仓库即可互相引用。
- 公共能力收敛在 `pkg` 模块（Postgres、NATS、Redis、gRPC 客户端、Kubernetes REST 客户端等），业务服务只依赖该模块而非直接依赖底层 SDK。
- `go.work.sum` 作为工作区级依赖锁文件，保证跨模块间接依赖一致。

### 3.2 Python 依赖：requirements.txt 精确锁定
- RAG 引擎使用 `requirements.txt` 以 `==` 或 `>=` 形式锁定依赖（如 `fastapi==0.139.0`、`llama-index-core==0.14.23`、`pymilvus>=2.5.0`）。
- CI 校验脚本的依赖单独维护在 `ci/requirements-contract.txt`，与业务代码解耦。
- 通过 `make test-python` 执行 `python -m compileall` 做语法检查，未集成 pipenv/poetry 等虚拟环境管理。

### 3.3 前端依赖：pnpm + overrides
- Console 前端使用 `pnpm`，`package.json` 中通过 `dependencies` 声明运行时依赖，`devDependencies` 声明构建期依赖。
- 使用 `overrides` 强制覆盖 `js-yaml`、`brace-expansion`、`postcss`、`minimatch`、`nanoid` 等传递依赖的版本，用于安全修复。
- API 类型通过 `openapi-typescript` 从 `api/openapi/services/v1.yaml` 和 Core OpenAPI 生成 TypeScript 类型，确保前端与后端契约同步。

### 3.4 代码生成驱动依赖
- Protobuf/gRPC：`Makefile` 中的 `tools-proto` 目标将 `protoc-gen-go`、`protoc-gen-go-grpc`、`protoc-gen-grpc-gateway` 安装到 `.bin/` 目录，`gen-proto` 通过 `buf generate` 在 `api/proto` 下生成 Go 代码。
- OpenAPI：`gen-api` 调用 `go generate ./...` 生成 Gateway 代码，并用 `openapi-typescript` 生成 Console 的 `schema.d.ts`。
- SDK：`gen-core-sdk` 调用 `scripts/gen_sdk_alpha.py` 从 Core/Services OpenAPI 生成 Go/Java/Python/TypeScript 四语 SDK。

### 3.5 私有镜像仓库与安全
- 镜像推送到私有 Harbor 仓库 `harbor.ani.internal/ani`，通过 GitHub Secrets 注入用户名密码。
- 构建缓存使用 `type=registry,ref=...:buildcache` 复用远程缓存。
- 镜像构建后自动执行 Trivy 扫描（阻断 HIGH/CRITICAL 漏洞）和 Cosign 签名，形成可验证的制品链。

## 4. 约定与约束

- **Go 模块边界**：新增 Go 服务必须在 `go.work` 的 `use` 块注册，否则 `make build` 无法发现该模块。
- **共享包依赖集中化**：PG/NATS/Redis/gRPC 等基础设施依赖应优先放入 `pkg/go.mod`，由服务通过 `replace` 引用，避免重复声明。
- **Python 依赖冻结**：生产环境部署时建议对 `requirements.txt` 使用 `pip freeze` 或 `pip-compile` 生成完整 lockfile，当前仓库仅用 `requirements.txt` 做最小版本约束。
- **前端依赖覆盖策略**：通过 `overrides` 强制升级已知漏洞的传递依赖，但应定期审计以避免破坏性变更。
- **构建工具本地化**：`protoc-gen-*` 等代码生成工具安装到 `./.bin/`，不污染全局 `GOPATH`，由 Makefile 统一管理版本。
- **CI 门禁**：`Makefile` 提供大量 `validate-*` 目标（如 `validate-services-contract`、`validate-openapi-spec`、`validate-core-alpha` 等），这些目标在 `make test` 中被串联执行，构成依赖与契约一致性门禁。
- **镜像制品链**：只有带 `v*.*.*` tag 的 push 或 workflow_dispatch 才会触发镜像构建，产物经 Trivy 扫描与 Cosign 签名后推送到 Harbor，形成可追溯的发布流程。