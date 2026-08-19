---
kind: frontend_style
name: 前端样式体系：基于 TDesign React + CSS Token 的 Console/BOSS 双端风格规范
category: frontend_style
scope:
    - '**'
source_files:
    - repo/frontends/console/package.json
    - repo/frontends/boss/package.json
    - repo/frontends/console/vite.config.ts
    - repo/frontends/console/src/main.tsx
    - repo/frontends/boss/src/main.tsx
    - repo/frontends/console/src/styles.css
    - repo/frontends/boss/src/styles.css
    - repo/frontends/console/src/components/shell/ConsolePage.tsx
    - repo/frontends/console/src/routes/_authenticated/index.tsx
    - docs/ANI-boss-email-notification-docs/references/UI规范/产品设计规范-TDesign组件与Token-2.0.md
---

## 1. 系统与方法论

ANI 平台的前端包含两个独立 Vite + React 应用：`repo/frontends/console`（租户控制台）与 `repo/frontends/boss`（运营后台）。两者共享同一套视觉与交互规范，统一基于 **TDesign React 1.x** 组件库实现，并通过其内置的 **CSS Design Token**（如 `--td-brand-color`、`--td-bg-color-page`、`--td-text-color-primary` 等）作为全局设计变量来源。

- 构建工具：Vite（console 使用 v8，boss 使用 v5），均通过 `@vitejs/plugin-react` 提供 React 支持。
- 路由：`@tanstack/react-router` 文件路由，生成 `src/routeTree.gen.ts`；boss 通过 `basepath: '/boss/'` 部署到子路径。
- 状态与数据：`@tanstack/react-query` 负责服务端数据缓存，`zustand` 管理客户端状态。
- 图表：`echarts` + `echarts-for-react`，色板须对齐 TDesign 语义色。
- 终端能力：`@xterm/xterm` + `@xterm/addon-fit` 用于实例观察页的 Terminal Tab。
- API 类型：通过 `openapi-typescript` 从 `../../api/openapi/services/v1.yaml` 生成 `schema.d.ts`，再经 `scripts/gen-core-schema.mjs` 生成 Core OpenAPI 类型，由 `openapi-fetch` 调用。

## 2. 关键文件

- `repo/frontends/console/package.json` / `repo/frontends/boss/package.json`：声明依赖与 `gen-api` 脚本。
- `repo/frontends/console/vite.config.ts`：TanStack Router 插件、`@` 别名、开发代理 `/api → http://localhost:8080`。
- `repo/frontends/console/src/main.tsx` / `boss/src/main.tsx`：入口，挂载 `QueryClientProvider`、`RouterProvider`，全局配置 `MessagePlugin.config({ placement: 'top', offset: [0, 16] })`，并引入 `tdesign-react/es/style/index.css`。
- `repo/frontends/console/src/styles.css` / `boss/src/styles.css`：全局基础样式，定义 body 背景 `#f2f3f3`、字体栈 `Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`，以及登录卡 `.auth-page` / `.auth-card` 样式，复用 `var(--td-bg-color-page)`、`var(--td-text-color-primary)` 等 Token。
- `repo/frontends/console/src/components/shell/ConsolePage.tsx`：页面级容器组件，以 flex 列布局 + `gap: 16px` 统一间距。
- `docs/ANI-boss-email-notification-docs/references/UI规范/产品设计规范-TDesign组件与Token-2.0.md`：UI 规范的权威文档，规定组件选型、Token 对照表、状态 Tag 映射、反模式等。

## 3. 架构与约定

### 3.1 组件与主题
- 所有 UI 组件必须来自 `tdesign-react`，图标来自 `tdesign-icons-react`，禁止混用 Ant Design、MUI、Element Plus 或自研 Button/Table。
- 颜色、背景、边框、文本一律通过 TDesign CSS 变量（Design Token）引用，禁止在页面中散落硬编码 hex 值。
- 按钮语义映射：Primary → `theme="primary"`，Secondary → `theme="default"`，Outline → `variant="outline"`，Text → `variant="text"`，Destructive → `theme="danger"`，Loading → `loading`。
- 状态 Tag 必须与后端 OpenAPI `status` 枚举一一对应，全站保持一致（success/warning/danger/default 等）。

### 3.2 布局与间距
- Console 壳层推荐结构：Header（品牌色 `--td-brand-color`）+ Aside（Menu，`theme="light"` 或 `dark`）+ Content（Breadcrumb + Page Header + 内容区）。
- 页面间距：`ConsolePage` 默认 `gap: 16px`；栅格 `Row gutter={16}`；壳层 Content padding 为 `20px 24px 32px`；卡片圆角统一为 4px。
- 登录页采用“Plain”风格：全屏居中、无 gamified 背景，参考 UX §4.1。

### 3.3 目录组织
- `src/routes/`：按 TanStack Router 文件路由组织页面，`_authenticated/` 下为鉴权后路由，`auth/callback.tsx` 处理 OIDC 回调。
- `src/components/shell/`：Console 页面基线组件（`ConsolePage`、`ConsoleContentCard`、`ConsolePageHeader`）。
- `src/features/instance-observability/`：按业务域划分的特性模块（EventsTab、LogsTab、MetricsTab、TerminalTab 等）。
- `src/api/`：OpenAPI 生成的类型与 `openapi-fetch` 客户端封装（`client.ts`、`coreClient.ts`、`auth.ts`）。
- `src/auth/session.ts`：会话持久化与 token 注入。

### 3.4 构建与开发
- 每个前端项目独立 `package.json`，通过 `pnpm-workspace.yaml` 组织。
- `dev` 脚本启动 Vite 开发服务器，`build` 执行生产构建，`preview` 预览产物。
- `gen-api` 脚本从根工程 `api/openapi/services/v1.yaml` 生成 TypeScript 类型，保证前后端契约一致。
- boss 应用通过 `basepath: '/boss/'` 部署于 Nginx/Apache 子路径，console 部署于根路径。

## 4. 约束与规则

以下规则来源于 UI 规范文档（`产品设计规范-TDesign组件与Token-2.0.md`）及代码实现，属于评审期可核查的约束：

1. **组件选型强制**：UI 组件必须使用 `tdesign-react`，图标必须使用 `tdesign-icons-react`；禁止引入 Ant Design、MUI、Element Plus 或自研平行 Button/Table。
2. **样式变量强制**：样式必须通过 TDesign Design Token（CSS 变量）实现，禁止 Tailwind 语义类、页面内散落 hex 颜色。
3. **布局壳层强制**：整体壳层需使用 TDesign `Layout`、`Menu`、`Breadcrumb`，不得完全自定义 shell 且不映射 TDesign。
4. **状态 Tag 一致性**：同一状态在不同页面必须使用相同颜色的 Tag，且与后端 OpenAPI `status` 字段对齐。
5. **列表页完整性**：列表页必须设计 loading / empty / error 态，否则评审驳回。
6. **危险操作保护**：删除等危险操作必须使用 `Dialog` 确认，仅 `Message` 提示视为反模式。
7. **主操作唯一性**：主操作区不允许出现多个 `primary` 按钮。
8. **图标无障碍**：图标若无文字说明，需提供 aria 描述。
9. **禁止复制源码**：禁止复制 TDesign 源码修改样式，也不得新建与 TDesign Button 并行的 `AniButton`（除非有文档化的扩展 variant）。
10. **图表色板限制**：ECharts 图表色板必须使用 TDesign 语义色（`--td-brand-color`、success/warning/error light 变体、`--td-component-border`），避免高饱和彩虹色。

这些规则在 UI 规范中以“禁止”“评审直接驳回”等措辞明确表达，并在 console/boss 两端的实际代码中被遵循（如登录卡样式、Tag theme 用法、Shell 布局等）。