---
kind: error_handling
name: ANI 平台错误处理体系：哨兵错误、ANIError 与网关统一响应
category: error_handling
scope:
    - '**'
source_files:
    - repo/pkg/types/errors.go
    - repo/pkg/ports/errors.go
    - repo/services/ani-gateway/internal/pkg/errors/errors.go
    - repo/services/ani-gateway/internal/middleware/request_id.go
    - repo/services/ani-gateway/internal/middleware/auth.go
    - repo/services/ani-gateway/internal/middleware/rbac.go
    - repo/services/ani-gateway/internal/middleware/ratelimit.go
    - repo/services/ani-gateway/internal/middleware/idempotency.go
    - repo/services/ani-gateway/internal/middleware/chain.go
    - repo/pkg/repo/task_repo.go
    - repo/pkg/adapters/registry/harbor_image_registry.go
---

## 1. 整体方案

ANI 平台采用「分层错误模型」：
- **领域层**（`pkg/types/errors.go`）定义跨服务共享的哨兵错误和结构化 `ANIError`，并维护 HTTP 状态码映射。
- **端口层**（`pkg/ports/errors.go`）为各能力适配器（Postgres、NATS、Registry、ObjectStore 等）暴露细粒度哨兵错误，用于被上层 adapter 通过 `errors.Is` 判断。
- **网关层**（`services/ani-gateway/internal/pkg/errors/errors.go` + `internal/middleware/request_id.go`）将内部错误转换为统一的 JSON 响应体 `{code, message, request_id, details}`，并通过中间件链在认证、鉴权、限流、幂等等环节直接输出标准错误。
- **业务/adapter 层**使用 `fmt.Errorf("...: %w", sentinel)` 或 `types.Wrapf(sentinel, ...)` 包装上下文信息，保留哨兵类型以便调用方用 `errors.Is` 判断。

该设计遵循 Go 标准库 `errors` 包约定：返回具体哨兵错误而非字符串，调用方通过 `errors.Is` 分支处理；结构化错误仅携带可安全返回给客户端的 `Code`/`Message`，`Cause` 仅服务端日志使用。

## 2. 关键文件与职责

| 文件 | 职责 |
|---|---|
| `repo/pkg/types/errors.go` | 定义全局哨兵错误（`ErrNotFound`、`ErrConflict`、`ErrUnauthorized`、`ErrForbidden`、`ErrBadRequest`、`ErrInvalidState`、`ErrLeaseTaken`、`ErrDeadLetter`）、`ANIError` 结构体、HTTP 状态码映射 `HTTPStatusForError`、以及 `Wrapf` 辅助函数 |
| `repo/pkg/ports/errors.go` | 定义能力适配器的细粒度哨兵错误（`ErrNotConfigured`、`ErrUnsupported`、`ErrNotFound`、`ErrConflict`、`ErrInvalid`、`ErrFailedPrecondition`、`ErrPayloadTooLarge`、`ErrInvalidCredentials`、`ErrTenantNotFound`、`ErrUnavailable` 及配额相关错误） |
| `repo/services/ani-gateway/internal/pkg/errors/errors.go` | 定义网关统一响应体 `APIError`、预定义错误码常量（`CodeNotFound`、`CodeUnauthorized`、`CodeForbidden`、`CodeBadRequest`、`CodeConflict`、`CodeInternalError`、`CodeRateLimitExceeded`、`CodeModelNotReady`），并提供 `RespondError`、`NotFound`、`Unauthorized` 快捷方法 |
| `repo/services/ani-gateway/internal/middleware/request_id.go` | 注入 `X-Request-ID`，并提供中间件专用的 `respondError` 写入 `{code, message, request_id}` 响应 |
| `repo/services/ani-gateway/internal/middleware/auth.go` | 认证失败时统一返回 `UNAUTHORIZED` / `FORBIDDEN` |
| `repo/services/ani-gateway/internal/middleware/rbac.go` | 鉴权失败时统一返回 `FORBIDDEN` |
| `repo/services/ani-gateway/internal/middleware/ratelimit.go` | 限流/存储不可用时返回 `RATE_LIMIT_EXCEEDED` / `RATE_LIMIT_UNAVAILABLE` |
| `repo/services/ani-gateway/internal/middleware/idempotency.go` | 幂等键冲突/过期/不可用时返回对应 `IDEMPOTENCY_*` 错误码 |
| `repo/pkg/repo/task_repo.go` | 典型使用 `types.Wrapf` 包装领域哨兵错误的示例 |
| `repo/pkg/adapters/registry/harbor_image_registry.go` | 典型使用 `errors.Is(err, ports.ErrNotFound)` 判断适配器哨兵错误的示例 |

## 3. 架构与约定

### 3.1 哨兵错误分层
- **领域级**（`pkg/types`）：面向所有 ANI 服务的通用语义错误，如“未找到”、“冲突”、“未授权”、“任务租约已被占用”。这些错误会被网关的 `HTTPStatusForError` 映射到 HTTP 4xx/5xx。
- **端口级**（`pkg/ports`）：面向具体能力适配器（数据库、对象存储、镜像仓库、NATS 等）的细粒度错误，便于 adapter 实现者精确区分“未配置”、“不支持”、“依赖不可用”等场景。

### 3.2 错误包装规范
- 使用 `fmt.Errorf("context: %w", sentinel)` 或 `types.Wrapf(sentinel, format, args...)` 包裹哨兵错误，保留底层哨兵类型，使调用方可用 `errors.Is` 判断。
- 禁止使用 `==` 比较错误，必须使用 `errors.Is`。
- 业务代码不直接构造 `ANIError`，而是返回哨兵错误，由网关层根据 `HTTPStatusForError` 或中间件逻辑决定最终 HTTP 状态码。

### 3.3 网关统一响应格式
所有 API 错误响应体遵循固定 JSON 结构：
```json
{
  "code": "NOT_FOUND",
  "message": "resource not found",
  "request_id": "req_abc123",
  "details": {}
}
```
- `code` 来自 `services/ani-gateway/internal/pkg/errors` 中定义的常量。
- `message` 是面向用户的可读消息。
- `request_id` 由 `RequestID` 中间件注入，贯穿整个请求链路。
- `details` 可选，用于附加结构化详情。

### 3.4 中间件错误处理链
Hertz 中间件注册顺序（`chain.go`）：`RequestID → Auth → RBAC → RateLimit → Idempotency → Audit → Route`。每个中间件在自身职责范围内直接调用 `respondError` 或 `RespondError` 输出标准错误并 `Abort()` 终止请求，避免进入下游 handler。

### 3.5 认证/鉴权/限流/幂等错误策略
- **认证失败**：返回 `UNAUTHORIZED`（包括 token 过期、无效、缺失）。
- **鉴权失败**：返回 `FORBIDDEN`（RBAC 拒绝、sandbox token 越权、租户上下文缺失）。
- **限流**：超过配额返回 `RATE_LIMIT_EXCEEDED`；限流存储不可用返回 `RATE_LIMIT_UNAVAILABLE`（503）。
- **幂等**：重复 key 返回 `IDEMPOTENCY_KEY_REUSED`；结果过期返回 `IdempotencyResultExpired`；进行中返回 `IDEMPOTENCY_IN_PROGRESS`；存储不可用返回 `IDEMPOTENCY_UNAVAILABLE`。

### 3.6 开发模式容错
当环境变量 `ANI_AUTH_MODE=dev` 时，认证中间件跳过真实鉴权，允许通过 `X-Dev-Tenant-ID` 和 `X-Dev-User-ID` 头模拟租户上下文，便于本地调试。

## 4. 约定与约束

- **所有 API 错误必须使用统一响应格式**：`services/ani-gateway/internal/pkg/errors/errors.go` 注释明确声明 “Every API error MUST use this format: {code, message, request_id, details}”。
- **错误码必须与 OpenAPI ErrorResponse 一致**：`pkg/types/errors.go` 注释要求 “every code here must appear in OpenAPI ErrorResponse”，保证契约一致性。
- **禁止使用 `==` 比较错误**：`pkg/types/errors.go` 注释明确要求 “use errors.Is() to check, never == comparison”。
- **适配器哨兵错误必须通过 `errors.Is` 判断**：adapter 测试（如 `harbor_image_registry_test.go`、`async_task_store_test.go`）大量使用 `errors.Is(err, ports.ErrNotFound)` 验证行为。
- **敏感信息不得泄露给客户端**：`ANIError.Cause` 字段注释说明 “underlying error, logged only”，仅服务端日志使用。
- **中间件 store 不能为空**：`chain.go` 中若 `GatewayStore` 为 nil 则 `panic`，确保限流/幂等功能在部署时必须正确配置。
- **请求追踪贯穿全链路**：`X-Request-ID` 由 RequestID 中间件注入，所有错误响应均携带该 ID，便于问题定位。

## 5. 与其他语言/模块的协作

- Python RAG 引擎（`repo/ai/rag-engine`）使用 FastAPI 原生异常，与 Go 侧通过 gRPC/HTTP 边界隔离，错误通过 gRPC status 或 HTTP 状态码传递。
- SDK 生成基于 OpenAPI 契约，错误码作为契约一部分被多语言 SDK 自动生成。
- CLI（`repo/cli/ani`）直接使用 `fmt.Errorf` 返回用户可见的错误信息，因为它是终端工具而非 API 服务。

## 6. 总结

ANI 平台的错误处理以「哨兵错误 + 结构化错误 + 网关统一响应」为核心，通过分层设计将领域语义、适配器细节与 HTTP 协议解耦。中间件集中处理横切错误（认证、鉴权、限流、幂等），业务代码专注于返回领域哨兵错误，由网关统一格式化输出。这种模式保证了错误的一致性、可观测性（request_id）和向后兼容性（OpenAPI 契约）。