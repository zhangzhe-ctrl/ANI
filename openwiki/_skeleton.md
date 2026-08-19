---
type: "Reference"
title: "ANI Wiki Skeleton (working file — delete after run)"
openwiki_generated: true
---

# ANI Wiki Skeleton (working file — delete after run)

Repository: KuberCloud ANI monorepo (`/repo`), an AI private-cloud platform split into
**ANI Core** (infrastructure control plane) and **ANI Services** (model/inference/KB business layer).
Working docs (`/ANI-*.md`, `/repo/CURRENT-SPRINT.md`) are scope/status sources, not code.

## Evidence summary (gathered)
- Gateway: `services/ani-gateway/main.go`, `internal/router/router.go` (RegisterOptions ~20 ports injected,
  Core `/api/v1/*`, Services transitional `/api/v1/svc/*`, OpenAI proxy `/v1/*`), `internal/middleware/chain.go`
  (RequestID→Auth→RBAC→RateLimit→Idempotency→Audit), runtime wiring files `*_runtime.go` reading env into
  `bootstrap.Config`.
- Ports/adapters: `pkg/ports/*.go` (workload_runtime, network_resources, storage_resources, gpu_inventory,
  quota, vector_store, secrets, encryption, image_registry, async_task, email_notification, sandbox_runtime,
  k8s_clusters, metering, observability, instance_observability, password_login, identity_provider,
  message_bus, cache_store, object_store, log_store, reconcile_controller, gpu_scheduling, gpu_spec,
  sandbox_template_catalog, metadata, health, errors); `pkg/adapters/runtime/*` (local_* + kubernetes_* +
  kubeovn + volcano + vcluster_helm + kms + loki + prometheus + storage_* + vector_store_*),
  `pkg/adapters/{postgres,redis,nats,registry,objectstore,vectorstore,metering,resilience,gpu,identity}`.
- Bootstrap: `pkg/bootstrap/{deps.go,server.go,instance.go,db.go,nats.go,redis.go,probes.go}` —
  `Config`, `Deps`, `Capabilities`, `MustConnect`, `RunGRPC`, `ConnectInstanceService`.
- gRPC services: auth-service (JWT/OIDC/API keys/refresh tokens/password login/platform login),
  task-service (async task repo, outbox publisher, NATS task consumer), metering-service (usage collectors,
  event consumer, rebuilder), model-service (Services model repo), tenant-service (tenant + plan + quota via
  Core client), reconcile-worker.
- Python: services/kb-service (gRPC 10 P0 RPCs + 3 P1 stubs, outbox dispatcher, session cache, Core API
  client, RLS repos, migrations); ai/rag-engine (FastAPI + gRPC Query + NATS parse worker, Milvus, embeddings).
- Frontends: console (TanStack router; compute/instances, gpu, gpu-containers, gpu-inventory, kb, models,
  registry, settings/api-keys, settings/gpu-queues, usage; auth login+callback; instance-observability feature),
  boss (login, auth/callback, ops/gpu-pool, integration/notification-settings/email {smtp,recipients,subscriptions}).
- Contracts: `api/openapi/v1.yaml` (Core), `api/openapi/services/v1.yaml` (Services), `api/proto/*`
  (auth/common/inference/kb/metering/model/task/tenant), `api/core-*` (alpha freeze, beta readiness,
  v1 compatibility baseline, contract changelogs), generated `pkg/generated/pb`, frontends `schema.d.ts`,
  `sdks/{core,services}/{go,python,typescript,java}`.
- Infra: `deploy/migrations` (RLS + resource tables), `deploy/helm/ani-platform` (profiles + component-contracts),
  `deploy/docker`, `deploy/real-k8s-lab`, installer profiles, Makefile validate-* targets + scripts/validate_*,
  `.github/workflows/build-image.yml`, `cli/ani`, `pkg/repo` (task/outbox), `pkg/types`, `pkg/security/sandboxtoken`,
  `pkg/nats/messages.go`, operators/inference-operator (CRD types only).

## Planned pages

### /openwiki/quickstart.md
Entrypoint: repo orientation (Core vs Services), high-level map, links to all sections, task-routing
table (change area → page → source entrypoints → focused tests → validation command), Backlog.

### /openwiki/architecture/overview.md
System context: Core/Services layering, request paths, gateway-centric topology, data stores
(Postgres/NATS/Redis/MinIO/Milvus/Harbor), deployment units. Mermaid flowchart of runtime topology.
Evidence: CLAUDE.md §3, README tree, main.go files.

### /openwiki/architecture/core-vs-services.md
Layer boundary rules: contract-only coupling, API split (`/api/v1` vs `/api/v1/svc`), SDK split,
import restrictions, controlled Services PR phase, CODEOWNERS/gates. Evidence: CLAUDE.md §3-5,
AGENTS.md, router.go svc group, Makefile validate-services.

### /openwiki/architecture/ports-and-adapters.md
Capability abstraction model: `pkg/ports` intent types (WorkloadRuntime, GPUInventory, etc.),
local vs real provider profiles, adapter registry, allowlist for direct component SDKs,
`bounded_direct` Kubernetes rule. Table mapping port → local adapter → real adapter.
Evidence: CLAUDE.md §5, pkg/ports/*.go, pkg/adapters/runtime/*.

### /openwiki/architecture/development-workflow.md
Sprint/batch discipline: CURRENT-SPRINT as source of truth, batch naming, development-records,
four-file update rule, guard series, validate-* gates, live gates, version policy.
Evidence: CLAUDE.md §6-7, ANI-12, Makefile targets, scripts/validate_*.

### /openwiki/gateway/overview.md
ani-gateway: Hertz server, main.go startup order and per-resource runtime config from env,
`bootstrap.Config`, health probes, graceful shutdown, gateway-owned stores (idempotency, quota).
Mermaid sequence of startup. Evidence: main.go, *_runtime.go, bootstrap/server.go, probes.go.

### /openwiki/gateway/middleware-chain.md
RequestID → Auth (JWT via auth-service, sandbox HMAC tokens, X-API-Key, ANI_AUTH_MODE=dev) →
RBAC → RateLimit → Idempotency (SetNX claim, fingerprint, 24h TTL, replay header) → Audit.
Mermaid sequence of request flow. Evidence: internal/middleware/*.go + tests.

### /openwiki/gateway/instances.md
`/api/v1/instances*` family: kinds (vm/container/gpu_container/sandbox/...), create flow
(plan→admission→dryrun→apply→store→async task), lifecycle ops (start/stop/restart/resize/...),
console sessions, snapshots, instance store implementations (memory/local/Postgres),
reconcile controller. Mermaid sequence of create. Evidence: router/instances.go,
pkg/adapters/runtime/instance_*.go, tests.

### /openwiki/gateway/network-storage-registry.md
`/api/v1/networks*`, `/volumes*`, `/filesystems*`, registry/Harbor endpoints: service/store/
renderer/dryrun/apply/status/reconciler pattern per resource family, Kube-OVN renderer,
Rook-Ceph storage control plane, Harbor + local image registry, shared in-process services
with instances. Evidence: network_resources.go, storage_resources.go, registry_resources.go,
pkg/adapters/runtime/{network,storage}_*.go, registry adapters.

### /openwiki/gateway/gpu.md
GPU inventory (discovery/classification via K8s node provider or local), GPU scheduling queues
(Volcano/local store), gpu-container instances, GPU specs service. `/api/v1/gpu-inventory`,
`/gpu-queues`, `/svc/gpu-containers`. Evidence: gpu_inventory_resources.go,
gpu_scheduling_resources.go, gpu_container_resources.go, ports/gpu_*.go, adapters.

### /openwiki/gateway/security-resources.md
Secrets, encryption (KMS/SM4), k8s-cluster proxy targets (mTLS), auth routes (token, OIDC
callback, password login, API keys), branding. Evidence: secret_resources.go,
encryption_resources.go, k8s_cluster_resources.go, auth.go, adapters kms/local/k8s secret.

### /openwiki/gateway/services-api.md
`/api/v1/svc/*` transitional routes: models, inference-services, knowledge-bases (gRPC client
to kb-service + SSE streaming), gpu-containers, sandboxes, tenant; `/v1/chat/completions`
OpenAI proxy; 501 stubs; 503 degradation when KB client nil. Evidence: router svc group,
kb_grpc_client.go, kb_sse.go, stubs.go, model/inference/sandbox/tenant resources.

### /openwiki/gateway/observability-quota-tasks.md
Observability endpoints (Prometheus proxy, Loki logs, instance metrics/events/terminal),
quota admin (Postgres quota store), async tasks API (202 + polling), metering endpoints,
email notifications. Evidence: observability.go, quota_resources.go, task_resources.go,
metering_resources.go, notifications_email_resources.go, instance_observability adapters.

### /openwiki/services/auth-service.md
gRPC auth: JWT issue/validate (RS256), OIDC (Dex) login + sessions + group-role map,
refresh tokens, token blocklist, API keys, password login (Postgres), platform login for BOSS.
Evidence: services/auth-service/internal/service/*.go, proto/auth.

### /openwiki/services/task-service.md
Async task control plane: TaskService gRPC (GetTask/UpdateTaskProgress), outbox pattern
(pkg/repo/outbox_repo + worker/outbox_publisher), NATS JetStream consumer for task mutation,
async_tasks migration. Evidence: task-service internals, pkg/repo, pkg/nats/messages.go.

### /openwiki/services/supporting-services.md
metering-service (usage collection, event consumer, rebuilder, metering_usage table),
model-service (Services model repository skeleton), tenant-service (tenant/plan stores,
Core quota client adapter, own ports/adapters), reconcile-worker (workload status reconcile
loop). Evidence: services/*/internal, metering adapters.

### /openwiki/services/kb-service.md
Python kb-service: gRPC server (10 P0 RPCs), dedicated asyncio loop bridging, repositories
(KB/document/chunk/message/async_task/outbox with RLS), outbox dispatcher to NATS parse
subject, session cache (Redis), CoreAPI client (vector-stores), migrations. Mermaid sequence
of document upload → outbox → rag-engine parse. Evidence: app/api/grpc_server.py,
app/repositories, app/outbox, app/core_api, migrations, tests.

### /openwiki/services/rag-engine.md
Python rag-engine: FastAPI lifespan wiring (Milvus init, embedding model, asyncpg pool),
gRPC Query RPC, NATS parse_worker (chunk/embed/index), retrieval modes, MinIO image upload.
Mermaid sequence of query flow. Evidence: ai/rag-engine/main.py, app/*, tests.

### /openwiki/frontends/console.md
Console app: React 18 + TDesign + TanStack Router/Query, generated Core/Services schema
clients, auth session (OIDC + password login tab), route map (compute/instances, gpu pages,
kb chat, models import, registry, settings, usage), instance-observability feature module.
Evidence: frontends/console/src routes, api, auth, features.

### /openwiki/frontends/boss.md
BOSS operations console: platform login, ops/gpu-pool, integration notification email
settings (smtp/recipients/subscriptions/test send) backed by gateway email notification APIs.
Evidence: frontends/boss/src routes, components/notification-email, api.

### /openwiki/contracts/openapi.md
Core `api/openapi/v1.yaml` + Services `services/v1.yaml`: URL prefix rules, auth schemes,
error format, cursor pagination, async task convention, idempotency_key rule, breaking-change
policy, alpha freeze/beta readiness/v1 compatibility baseline files, changelogs, mock server.
Evidence: yaml headers, api/core-*, scripts serve_core_mock.

### /openwiki/contracts/grpc-and-sdks.md
`api/proto` layout + buf generation → `pkg/generated/pb`; internal-only gRPC rule;
four-language Core/Services SDKs, generation targets (gen-core-sdk), SDK metadata, CLI `ani`
as thin REST client; frontend schema generation (gen-console-api).
Evidence: api/proto, buf.gen.yaml, sdks/*, cli/ani/main.go, Makefile.

### /openwiki/data/database.md
Postgres schema: deploy/migrations history (init, idempotency ops, permissions, network/
storage resources, API keys, cluster proxy targets, leases, refresh tokens/users, metering,
async tasks, storage control plane), RLS tenancy (`WithTenantTx`/`SetDBTenant`), kb-service
migrations. erDiagram of principal tables. Evidence: deploy/migrations, pkg/adapters/postgres,
pkg/repo, rls_prerequisite_test.go.

### /openwiki/operations/deployment.md
Helm umbrella chart (profiles + component-contracts), docker-compose local dev, env config
(.env.example → bootstrap.Config), installer profiles, real-k8s-lab, CI build-image workflow,
release process, start.sh. Evidence: deploy/*, installer profiles, Makefile, workflows.

### /openwiki/operations/validation-gates.md
Gate taxonomy: unit tests, validate-architecture, contract validators (alpha/beta/baseline),
service boundary gates, live gates (real provider), doc entrypoint checks; how to run
(focused vs broad). Evidence: Makefile, scripts/validate_*, CLAUDE.md §6.

## Notes
- services/prototypes, services/ani-services.html, conversation_history, .qoder, .gitnexus:
  generated/prototype artifacts, not runtime — mention only in quickstart scope boundaries.
- operators/inference-operator contains only CRD API types → cover in contracts/grpc page
  briefly or overview; not a full page.
- Root /ANI-*.md are product/design docs — reference as intent sources, do not duplicate.
