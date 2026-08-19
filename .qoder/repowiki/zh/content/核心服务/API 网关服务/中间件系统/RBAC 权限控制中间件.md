# RBAC 权限控制中间件

<cite>
**本文引用的文件**
- [rbac.go](file://repo/services/ani-gateway/internal/middleware/rbac.go)
- [auth.go](file://repo/services/ani-gateway/internal/middleware/auth.go)
- [auth_client.go](file://repo/services/ani-gateway/internal/middleware/auth_client.go)
- [auth_service.go](file://repo/services/auth-service/internal/service/auth_service.go)
- [jwt.go](file://repo/services/auth-service/internal/service/jwt.go)
- [config.go](file://repo/services/auth-service/internal/config/config.go)
- [rbac_test.go](file://repo/services/ani-gateway/internal/middleware/rbac_test.go)
- [auth_test.go](file://repo/services/ani-gateway/internal/middleware/auth_test.go)
- [apply_kb_migration.py](file://repo/scripts/apply_kb_migration.py)
- [rls.py](file://repo/services/kb-service/app/repositories/rls.py)
</cite>

## 目录
1. [简介](#简介)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构总览](#架构总览)
5. [详细组件分析](#详细组件分析)
6. [依赖关系分析](#依赖关系分析)
7. [性能与缓存](#性能与缓存)
8. [故障排查指南](#故障排查指南)
9. [结论](#结论)
10. [附录：配置与集成示例](#附录配置与集成示例)

## 简介
本文件面向 ANI 平台的 RBAC（基于角色的访问控制）权限控制中间件，系统性说明其实现原理、权限模型、资源访问控制流程、动态权限评估、缓存机制与性能优化策略，并给出配置选项、权限继承、多租户隔离以及与认证系统的集成方式。文档同时提供流程图与时序图，帮助读者快速理解从请求进入网关到最终授权决策的完整链路。

## 项目结构
RBAC 能力由“网关鉴权 + 网关 RBAC 中间件 + 认证服务”三部分协作完成：
- 网关鉴权中间件负责解析令牌、校验作用域、注入租户上下文。
- RBAC 中间件根据 HTTP 方法与路径推断资源与动作，调用认证服务的 CheckPermission 进行授权判定。
- 认证服务实现 JWT/API Key 校验、角色与作用域匹配、内置角色规则（平台管理员、租户管理员、审计员、普通用户等）。

```mermaid
graph TB
Client["客户端"] --> GW["ANI Gateway<br/>鉴权+RBAC中间件"]
GW --> AS["Auth Service<br/>gRPC 鉴权与授权"]
AS --> DB["PostgreSQL<br/>JWT黑名单/凭据存储"]
AS --> Cache["Redis<br/>令牌黑名单/限流缓存"]
GW --> Store["业务存储<br/>RLS 行级安全(可选)"]
```

图表来源
- [auth.go:18-142](file://repo/services/ani-gateway/internal/middleware/auth.go#L18-L142)
- [rbac.go:14-72](file://repo/services/ani-gateway/internal/middleware/rbac.go#L14-L72)
- [auth_service.go:123-187](file://repo/services/auth-service/internal/service/auth_service.go#L123-L187)

章节来源
- [auth.go:18-142](file://repo/services/ani-gateway/internal/middleware/auth.go#L18-L142)
- [rbac.go:14-72](file://repo/services/ani-gateway/internal/middleware/rbac.go#L14-L72)
- [auth_service.go:123-187](file://repo/services/auth-service/internal/service/auth_service.go#L123-L187)

## 核心组件
- 网关鉴权中间件
  - 职责：解析 Bearer/JWT、API Key、沙箱令牌；校验作用域；设置 tenant_id/user_id/roles/scope；将租户上下文注入 Go context。
  - 关键点：开发模式开关、公开路径白名单、作用域路由隔离（platform/tenant/sandbox）。
- RBAC 中间件
  - 职责：推断资源与动作；调用 auth-service 的 CheckPermission；根据返回结果放行或拒绝。
  - 关键点：方法到动作映射、路径到资源推断、sandbox 特殊处理、错误统一响应。
- 认证服务
  - 职责：JWT 验证、API Key 校验、刷新令牌、令牌撤销、CheckPermission 授权逻辑。
  - 关键点：内置角色与规则（平台管理员、租户管理员、审计员、普通用户）、作用域匹配、读操作限制。

章节来源
- [auth.go:18-142](file://repo/services/ani-gateway/internal/middleware/auth.go#L18-L142)
- [rbac.go:14-72](file://repo/services/ani-gateway/internal/middleware/rbac.go#L14-L72)
- [auth_service.go:123-187](file://repo/services/auth-service/internal/service/auth_service.go#L123-L187)

## 架构总览
下图展示一次受保护 API 调用的端到端流程：客户端携带令牌到达网关，鉴权中间件解析并注入上下文，RBAC 中间件推断资源与动作后调用认证服务进行授权判定，最终决定是否执行业务逻辑。

```mermaid
sequenceDiagram
participant C as "客户端"
participant G as "Gateway 鉴权中间件"
participant R as "Gateway RBAC 中间件"
participant A as "Auth Service"
participant S as "业务处理器"
C->>G : "HTTP 请求(Bearer/API Key)"
G->>G : "校验令牌/作用域/注入上下文"
G-->>R : "tenant_id, user_id, roles, scope"
R->>R : "推断 resource/action"
R->>A : "CheckPermission(tenant,user,roles,resource,action)"
A-->>R : "Allowed/Reason"
alt 允许
R-->>S : "继续处理"
S-->>C : "成功响应"
else 拒绝
R-->>C : "403 FORBIDDEN"
end
```

图表来源
- [auth.go:18-142](file://repo/services/ani-gateway/internal/middleware/auth.go#L18-L142)
- [rbac.go:14-72](file://repo/services/ani-gateway/internal/middleware/rbac.go#L14-L72)
- [auth_service.go:123-187](file://repo/services/auth-service/internal/service/auth_service.go#L123-L187)

## 详细组件分析

### 网关鉴权中间件（Auth）
- 令牌类型支持
  - Bearer JWT：本地解析签名、有效期、签发者、黑名单；提取 tenant_id/user_id/roles/scope。
  - API Key：以特定前缀识别，走同一 ValidateToken 流程，默认 tenant 作用域。
  - 沙箱令牌：HMAC 本地校验，仅允许访问实例沙箱子资源。
- 作用域隔离
  - platform：仅可访问 /api/v1/auth/platform/*、/api/v1/platform/*、/api/v1/admin/*。
  - tenant：其他路由。
  - sandbox：仅 /api/v1/instances/{id}/sandbox/*。
- 上下文注入
  - 在 RequestContext 中设置 tenant_id/user_id/roles/scope。
  - 通过 types.TenantContext 注入 Go context，供后续存储层使用（如 RLS）。

```mermaid
flowchart TD
Start(["进入 Auth"]) --> Public{"是否公开路径?"}
Public -- 是 --> Next1["直接放行"]
Public -- 否 --> Mode{"是否开发模式?"}
Mode -- 是 --> DevCtx["注入开发租户/用户/角色"]
Mode -- 否 --> Token{"Bearer/API Key/Sandbox?"}
Token -- Bearer --> ValidateJWT["校验JWT/作用域"]
Token -- API Key --> ValidateKey["校验API Key/作用域"]
Token -- Sandbox --> ValidateSB["校验沙箱令牌/作用域"]
ValidateJWT --> SetCtx["设置上下文"]
ValidateKey --> SetCtx
ValidateSB --> SetCtx
SetCtx --> Next2["放行"]
DevCtx --> Next2
Next1 --> End(["结束"])
Next2 --> End
```

图表来源
- [auth.go:18-142](file://repo/services/ani-gateway/internal/middleware/auth.go#L18-L142)
- [auth.go:214-229](file://repo/services/ani-gateway/internal/middleware/auth.go#L214-L229)

章节来源
- [auth.go:18-142](file://repo/services/ani-gateway/internal/middleware/auth.go#L18-L142)
- [auth.go:214-229](file://repo/services/ani-gateway/internal/middleware/auth.go#L214-L229)

### RBAC 中间件（RBAC）
- 资源与动作推断
  - 资源：从路径中提取 v1 后的第一段（若为 svc 则取下一段）。
  - 动作：GET/HEAD→get，POST→create，PUT/PATCH→update，DELETE→delete，其他→小写方法名。
- 授权流程
  - 跳过公开路径与开发模式。
  - 校验租户上下文与作用域（sandbox 特殊处理）。
  - 调用 auth-service 的 CheckPermission，依据 Allowed/Reason 决定放行或拒绝。
- 错误处理
  - 统一返回 403 FORBIDDEN，包含原因信息。

```mermaid
flowchart TD
Start(["进入 RBAC"]) --> Public{"是否公开路径?"}
Public -- 是 --> Next["放行"]
Public -- 否 --> Dev{"是否开发模式?"}
Dev -- 是 --> Next
Dev -- 否 --> Ctx{"是否有租户上下文/作用域合法?"}
Ctx -- 否 --> Deny["403 FORBIDDEN"]
Ctx -- 是 --> Infer["推断 resource/action"]
Infer --> Call["调用 CheckPermission"]
Call --> Allowed{"是否允许?"}
Allowed -- 是 --> Next
Allowed -- 否 --> Deny
```

图表来源
- [rbac.go:14-72](file://repo/services/ani-gateway/internal/middleware/rbac.go#L14-L72)
- [rbac.go:74-98](file://repo/services/ani-gateway/internal/middleware/rbac.go#L74-L98)

章节来源
- [rbac.go:14-72](file://repo/services/ani-gateway/internal/middleware/rbac.go#L14-L72)
- [rbac.go:74-98](file://repo/services/ani-gateway/internal/middleware/rbac.go#L74-L98)

### 认证服务（Auth Service）
- 令牌校验
  - JWT：RS256 签名校验、过期时间、签发者、黑名单检查。
  - API Key：前缀识别、速率限制、返回 tenant 上下文。
- 授权规则（CheckPermission）
  - 平台管理员/租户管理员：直接允许。
  - 作用域匹配：支持精确匹配、资源级通配、全局通配。
  - 审计员：仅允许读操作（get/list/read/watch）。
  - 普通用户：允许有限动作（get/list/read/watch/use/create）。
  - 未命中任何规则：拒绝并返回原因。

```mermaid
flowchart TD
Start(["CheckPermission"]) --> Valid{"参数有效?"}
Valid -- 否 --> Deny["拒绝(缺少参数)"]
Valid -- 是 --> Admin{"是否平台/租户管理员?"}
Admin -- 是 --> Allow["允许"]
Admin -- 否 --> Scope{"是否匹配作用域?"}
Scope -- 是 --> Allow
Scope -- 否 --> Auditor{"是否审计员且读操作?"}
Auditor -- 是 --> Allow
Auditor -- 否 --> User{"是否普通用户且允许动作?"}
User -- 是 --> Allow
User -- 否 --> Deny2["拒绝(无匹配)"]
```

图表来源
- [auth_service.go:161-187](file://repo/services/auth-service/internal/service/auth_service.go#L161-L187)
- [auth_service.go:232-242](file://repo/services/auth-service/internal/service/auth_service.go#L232-L242)
- [auth_service.go:261-277](file://repo/services/auth-service/internal/service/auth_service.go#L261-L277)

章节来源
- [auth_service.go:123-187](file://repo/services/auth-service/internal/service/auth_service.go#L123-L187)
- [auth_service.go:232-242](file://repo/services/auth-service/internal/service/auth_service.go#L232-L242)
- [auth_service.go:261-277](file://repo/services/auth-service/internal/service/auth_service.go#L261-L277)

### 多租户隔离与行级安全（RLS）
- 网关侧
  - 鉴权中间件将 tenant_id 注入 Go context，供存储层使用。
- 数据库侧
  - PostgreSQL 启用 RLS，创建 tenant_isolation 策略，按 current_setting('app.current_tenant_id') 过滤数据。
  - Python 服务通过 SET LOCAL app.current_tenant_id 在事务内设置租户上下文，确保所有 SQL 受 RLS 约束。

```mermaid
graph LR
Ctx["Go Context<br/>TenantContext"] --> Tx["事务/连接"]
Tx --> Set["SET LOCAL app.current_tenant_id"]
Set --> PG["PostgreSQL RLS 策略"]
PG --> Filter["按 tenant_id 过滤行"]
```

图表来源
- [auth.go:160-197](file://repo/services/ani-gateway/internal/middleware/auth.go#L160-L197)
- [apply_kb_migration.py:125-180](file://repo/scripts/apply_kb_migration.py#L125-L180)
- [rls.py:1-32](file://repo/services/kb-service/app/repositories/rls.py#L1-L32)

章节来源
- [auth.go:160-197](file://repo/services/ani-gateway/internal/middleware/auth.go#L160-L197)
- [apply_kb_migration.py:125-180](file://repo/scripts/apply_kb_migration.py#L125-L180)
- [rls.py:1-32](file://repo/services/kb-service/app/repositories/rls.py#L1-L32)

## 依赖关系分析
- 网关中间件依赖
  - 认证客户端：用于 ValidateToken 与 CheckPermission。
  - 沙箱令牌库：用于本地 HMAC 校验与范围判断。
  - 租户上下文工具：用于注入 Go context。
- 认证服务依赖
  - JWT 验证器：RS256 签名校验、黑名单检查。
  - 存储与缓存：PostgreSQL（凭据、黑名单）、Redis（黑名单、限流）。
  - OIDC/密码登录管理器：登录流程与组映射。

```mermaid
classDiagram
class GatewayAuth {
+ValidateToken()
+SetTenantContext()
}
class GatewayRBAC {
+InferPermission()
+CheckPermission()
}
class AuthService {
+ValidateToken()
+CheckPermission()
+RevokeToken()
}
class JWTValidator {
+Validate()
}
class Storage {
+PostgreSQL
+Redis
}
GatewayAuth --> AuthService : "gRPC"
GatewayRBAC --> AuthService : "gRPC"
AuthService --> JWTValidator : "使用"
AuthService --> Storage : "读写"
```

图表来源
- [auth.go:18-142](file://repo/services/ani-gateway/internal/middleware/auth.go#L18-L142)
- [rbac.go:14-72](file://repo/services/ani-gateway/internal/middleware/rbac.go#L14-L72)
- [auth_service.go:21-57](file://repo/services/auth-service/internal/service/auth_service.go#L21-L57)
- [jwt.go:21-94](file://repo/services/auth-service/internal/service/jwt.go#L21-L94)

章节来源
- [auth.go:18-142](file://repo/services/ani-gateway/internal/middleware/auth.go#L18-L142)
- [rbac.go:14-72](file://repo/services/ani-gateway/internal/middleware/rbac.go#L14-L72)
- [auth_service.go:21-57](file://repo/services/auth-service/internal/service/auth_service.go#L21-L57)
- [jwt.go:21-94](file://repo/services/auth-service/internal/service/jwt.go#L21-L94)

## 性能与缓存
- 令牌黑名单缓存
  - 认证服务使用 Redis 维护 JWT 黑名单键，校验时快速判断是否被撤销。
  - 撤销接口将 JTI 写入缓存并设置 TTL，避免频繁查库。
- 会话缓存（kb-service）
  - Redis 作为最佳努力缓存，失败不阻断主流程，降级为仅数据库持久化。
  - 使用 pipeline 批量执行 RPUSH/EXPIRE/LTRIM，减少网络往返。
- 网关侧建议
  - 对 CheckPermission 调用增加本地短生命周期缓存（按 tenant/user/resource/action），注意失效策略（角色变更、令牌更新）。
  - 合理设置超时与重试，避免认证服务抖动影响整体吞吐。

章节来源
- [auth_service.go:110-121](file://repo/services/auth-service/internal/service/auth_service.go#L110-L121)
- [jwt.go:149-170](file://repo/services/auth-service/internal/service/jwt.go#L149-L170)

## 故障排查指南
- 常见错误码与原因
  - 401 UNAUTHORIZED：令牌无效/过期、API Key 无效、认证服务不可用。
  - 403 FORBIDDEN：作用域不匹配、RBAC 拒绝、租户上下文缺失。
- 定位步骤
  - 确认请求头 Authorization/X-API-Key 是否正确。
  - 检查作用域是否与路由匹配（platform/tenant/sandbox）。
  - 查看 RBAC 推断的资源与动作是否符合预期。
  - 核对认证服务返回的 Allowed/Reason。
- 日志与测试
  - 使用单元测试覆盖 inferPermission、scopeAllowedForPath 等关键函数。
  - 在开发模式下通过 X-Dev-Tenant-ID/X-Dev-User-ID 模拟租户上下文。

章节来源
- [auth.go:18-142](file://repo/services/ani-gateway/internal/middleware/auth.go#L18-L142)
- [rbac.go:14-72](file://repo/services/ani-gateway/internal/middleware/rbac.go#L14-L72)
- [auth_test.go:38-66](file://repo/services/ani-gateway/internal/middleware/auth_test.go#L38-L66)
- [rbac_test.go:5-25](file://repo/services/ani-gateway/internal/middleware/rbac_test.go#L5-L25)

## 结论
本 RBAC 中间件通过网关鉴权与授权分离、作用域隔离、内置角色规则与数据库 RLS 多层防护，实现了清晰、可扩展且高性能的权限控制体系。结合令牌黑名单与最佳努力缓存，系统在可用性与性能之间取得良好平衡。未来可引入 OPA 等外部策略引擎，进一步增强动态策略评估能力。

## 附录：配置与集成示例
- 环境变量（认证服务）
  - DATABASE_URL、NATS_URL、REDIS_URL、GRPC_PORT、HEALTH_PORT
  - AUTH_JWT_PUBLIC_KEY_PEM/AUTH_JWT_PRIVATE_KEY_PEM、AUTH_JWT_ISSUER
  - AUTH_OIDC_* 系列 OIDC 相关配置
- 网关开发模式
  - 设置 ANI_AUTH_MODE=dev，并通过 X-Dev-Tenant-ID/X-Dev-User-ID 注入上下文。
- 作用域与路由
  - platform：/api/v1/auth/platform/*、/api/v1/platform/*、/api/v1/admin/*
  - tenant：其余路由
  - sandbox：/api/v1/instances/{id}/sandbox/*
- 权限规则示例（概念）
  - 资源:动作 精确匹配（如 tasks:get）
  - 资源:* 资源级通配（如 tasks:*）
  - *: * 全局通配
  - scope:资源:动作 作用域限定匹配
- 多租户隔离
  - 在事务内设置 app.current_tenant_id，配合 PostgreSQL RLS 策略实现行级隔离。

章节来源
- [config.go:10-52](file://repo/services/auth-service/internal/config/config.go#L10-L52)
- [auth.go:214-229](file://repo/services/ani-gateway/internal/middleware/auth.go#L214-L229)
- [apply_kb_migration.py:125-180](file://repo/scripts/apply_kb_migration.py#L125-L180)
- [rls.py:1-32](file://repo/services/kb-service/app/repositories/rls.py#L1-L32)