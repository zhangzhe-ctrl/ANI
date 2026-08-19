---
kind: logging_system
name: 基于 Go slog JSON 与 Python logging 的双栈结构化日志系统
category: logging_system
scope:
    - '**'
source_files:
    - repo/pkg/bootstrap/server.go
    - repo/pkg/bootstrap/deps.go
    - repo/services/ani-gateway/main.go
    - repo/pkg/adapters/nats/message_bus.go
    - repo/ai/rag-engine/main.py
    - repo/ai/rag-engine/app/grpc/server.py
    - repo/services/kb-service/main.py
---

## 1. 使用的系统与框架

仓库采用**双语言、双框架**的日志体系：

- **Go 服务（Core/Services）**：统一使用标准库 `log/slog`，以 JSON 格式输出到 stdout。所有 Go 进程在启动时通过 `slog.NewJSONHandler(os.Stdout, nil)` 创建 handler，并调用 `slog.SetDefault(logger)` 设置全局默认 logger。
- **Python RAG Engine / KB Service**：使用 Python 标准库 `logging`，按模块名获取 logger（`logging.getLogger(__name__)`），RAG Engine 入口通过 `logging.basicConfig(level=logging.INFO)` 配置根处理器；KB Service 通过 Uvicorn 的 `log_level="info"` 参数控制。

没有引入第三方日志库（如 zap、logrus、structlog、loguru），全部依赖语言内置能力。

## 2. 关键文件与位置

| 组件 | 关键文件 | 职责 |
|---|---|---|
| Go 共享引导 | `repo/pkg/bootstrap/server.go` | 创建 JSON handler、设置 `slog.Default()`、提供 gRPC 拦截器 |
| Go 网关入口 | `repo/services/ani-gateway/main.go` | 每个服务 main 中初始化 JSON logger |
| Go 依赖注入 | `repo/pkg/bootstrap/deps.go` | `Deps.Logger *slog.Logger` 字段，供各适配器消费 |
| NATS 适配器 | `repo/pkg/adapters/nats/message_bus.go` | 接收 `*slog.Logger` 作为构造参数 |
| RAG Engine 入口 | `repo/ai/rag-engine/main.py` | FastAPI lifespan 中用 `logger.info/warning` 记录启动/关闭 |
| RAG gRPC 服务器 | `repo/ai/rag-engine/app/grpc/server.py` | `basicConfig(level=logging.INFO)` 配置根日志 |
| KB Service | `repo/services/kb-service/main.py` | Uvicorn `log_level="info"` |

## 3. 架构与约定

### 3.1 Go 侧：集中式 JSON 日志 + 上下文传播

- **初始化点唯一**：`bootstrap.MustConnect`（server.go:110-111）和每个服务 main（如 ani-gateway:21-22）均执行 `slog.NewJSONHandler(os.Stdout, nil)` + `slog.SetDefault(logger)`，保证所有 `slog.Info/Warn/Error` 输出为 JSON 行。
- **级别策略**：handler 未设置 Level 选项，即使用 `slog` 默认级别（INFO）。代码中仅出现 `slog.Info`、`slog.Warn`、`slog.Error`，未见 `Debug` 或自定义级别。
- **结构化字段**：所有日志通过 key-value 对传递上下文，例如：
  - `slog.Info("database connected", "version", version)`
  - `slog.Warn("quota Confirm: reservation not in reserved state, skipping")`
  - `slog.Error("failed to connect to database", "err", err)`
- **gRPC 拦截器**：`loggingUnaryInterceptor` 自动记录每个 RPC 的 method 与错误；`recoveryUnaryInterceptor` 捕获 panic 并以 `ErrorContext` 输出。
- **依赖注入**：`Deps` 结构体持有 `Logger *slog.Logger`，NATS MessageBus 等适配器通过构造函数显式接收 logger，避免直接依赖全局默认值。
- **优雅停机**：`RunGRPC` / `RunHealthProbe` 在 SIGINT/SIGTERM 后输出 `shutting down` / `stopped` 日志。

### 3.2 Python 侧：模块级 logger + Uvicorn 根处理器

- RAG Engine 每个模块顶部 `logger = logging.getLogger(__name__)`，通过相对模块路径区分来源。
- 启动阶段通过 `logging.basicConfig(level=logging.INFO)` 配置根处理器（仅在 `__main__` 分支生效）。
- KB Service 通过 Uvicorn 启动参数 `log_level="info"` 控制级别。
- 测试中使用 `caplog.at_level("WARNING", logger="app.services.summary_service")` 进行断言。

### 3.3 前端日志查看

前端控制台 `frontends/console/src/features/instance-observability/LogsTab.tsx` 提供实例观测中的日志过滤 UI（包含 levelFilter），用于展示后端产生的实例日志。

## 4. 约定与约束

- **必须输出 JSON**：Go 服务统一使用 `NewJSONHandler(os.Stdout, nil)`，便于 Kubernetes 日志采集系统（如 Loki/Fluent Bit）解析。
- **禁止裸 fmt.Println**：所有 Go 业务日志应通过 `slog.Info/Warn/Error`，而非 `fmt.Print*` 或 `log.Print*`。
- **级别使用规范**：仅使用 INFO/WARN/ERROR 三级；未发现 DEBUG 级别的使用，也未见通过环境变量动态调整 log level 的逻辑。
- **结构化字段命名**：错误信息统一使用 `"err"` 键；方法名使用 `"method"`；端口使用 `"port"`；数据库版本使用 `"version"` 等，保持跨模块一致。
- **上下文传播**：gRPC 路径使用 `logger.ErrorContext(ctx, ...)` / `InfoContext(ctx, ...)` 携带 context；普通路径直接使用 `slog.Xxx`。
- **Python 模块隔离**：每个 Python 模块独立 `getLogger(__name__)`，不共享全局 logger 实例，便于按模块过滤。
- **健康探针日志**：health probe 启动/停止均有对应日志，便于容器编排层判断生命周期。

## 5. 已知缺口

- 无统一的 `LOG_LEVEL` 环境变量驱动机制（Go 侧 handler 未配置 Level，Python 侧各自硬编码）。
- 无 trace/correlation ID 贯穿请求链路（gRPC interceptor 仅记录 method 与 error，未提取/注入 request id）。
- 无日志轮转/文件大小限制（stdout 直写，依赖外部日志收集器）。
- 前端 LogsTab 仅做 UI 过滤，实际日志源由后端实例观测能力提供，不在本仓库内实现。