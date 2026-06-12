# OpenClaw 代码库技术分析报告

> 分析对象：`openclaw/openclaw` monorepo（版本 2026.6.2，MIT 协议）
> 分析日期：2026-06-12

---

## 1. 项目概述

**OpenClaw** 是一个运行在用户自有设备上的**个人 AI 助手 / 多通道 AI 网关**（"Multi-channel AI gateway with extensible messaging integrations"）。其核心价值主张：

- 在用户已有的聊天通道上响应用户：支持 WhatsApp、Telegram、Slack、Discord、Signal、iMessage、Microsoft Teams、Matrix、飞书、LINE、QQ、Zalo、WebChat 等 20+ 通道；
- 可在 macOS/iOS/Android 上语音对话，并渲染可控的实时 Canvas；
- Gateway 仅是控制平面，产品本体是"助手"——单用户、本地优先、长驻运行；
- 定位为"真正能做事的 AI"：在真实计算机上执行真实任务（命令执行、浏览器、工具调用、MCP）。

项目演化路径：Warelay → Clawdbot → Moltbot → OpenClaw。当前优先级（见 `VISION.md`）：安全与安全默认值、Bug 修复与稳定性、安装/首跑体验，其次是全模型供应商支持、主流消息通道增强、性能与测试基础设施、computer-use 能力、各平台伴生应用。

## 2. 代码库规模

| 指标 | 数值 |
|---|---|
| Git 跟踪文件总数 | ~19,900 |
| TypeScript 文件（.ts/.tsx） | ~16,200 |
| 核心 `src/`（不含测试） | ~23.5 万行 |
| 插件 `extensions/`（不含测试） | ~11 万行 |
| 共享包 `packages/`（不含测试） | ~4.6 万行 |
| 测试文件（`*.test.ts`） | 6,265 个 |
| 插件（extensions）数量 | 143 个 |
| 共享 workspace 包 | 21 个 |
| 根 `package.json` scripts | 444 条 |
| 运行时依赖（root） | 55 个 |
| CI 工作流 | 59 个 |

这是一个**超大规模 TypeScript ESM monorepo**，pnpm workspace 管理四个工作区：根（核心）、`ui/`、`packages/*`、`extensions/*`。

## 3. 总体架构

```
                  ┌────────────────────────────────────────────┐
   消息通道插件 ──▶│  Gateway (WebSocket/HTTP 控制平面, 协议 v4)  │◀── 原生应用/UI/SDK 客户端
 (telegram/slack/ │   src/gateway/  packages/gateway-protocol  │
  discord/...)    └──────────────────┬─────────────────────────┘
                                     │ RPC 方法分发
                  ┌──────────────────▼─────────────────────────┐
                  │  路由 + 自动回复  src/routing/ src/auto-reply│
                  └──────────────────┬─────────────────────────┘
                  ┌──────────────────▼─────────────────────────┐
                  │  Agent 运行时  src/agents/                  │
                  │  会话/运行/工具调用/MCP/LSP/Bash/模型回退     │
                  └───────┬──────────────────────┬─────────────┘
                          │                      │
              ┌───────────▼──────────┐  ┌────────▼──────────────┐
              │ LLM Provider 插件     │  │ SQLite 状态存储 (Kysely)│
              │ (anthropic/openai/...)│  │ src/state/            │
              └──────────────────────┘  └───────────────────────┘
```

架构核心原则（在根 `AGENTS.md` 中以硬性策略固化）：

1. **核心与插件解耦**：核心保持 plugin-agnostic，不内置插件 id/默认值/策略；插件只能通过 `openclaw/plugin-sdk/*`、manifest 元数据、注入的 runtime helper 和文档化 barrel（`api.ts`、`runtime-api.ts`）进入核心。
2. **单一规范路径**：运行时只读取规范配置/数据形态，旧形态只在 `openclaw doctor --fix` 迁移代码中处理，不留运行时兼容垫片。
3. **存储默认 SQLite**：OpenClaw 自有运行时状态一律 SQLite（Kysely 类型安全查询），禁止新增 JSON/JSONL/sidecar 文件存放状态。
4. **传输层纯粹性**：消息通道插件只做传输（transport-only），渲染可移植的展示/动作、执行传输限制、映射原生回调，不拥有产品命令树或业务策略。

## 4. 核心运行时（`src/`）

### 4.1 入口与引导

- `src/entry.ts`：进程入口，处理 compile-cache respawn（`src/entry.respawn.ts`）、`--version`/`--help` 快速路径（`src/entry.version-fast-path.ts` 等），尽量绕过昂贵的 CLI 全量加载——对一个常驻 CLI 来说启动延迟是重点优化对象。
- `src/cli/run-main.ts`：CLI 主编排——加载 `.env`、引导网络代理生命周期、构建 Commander 程序、**懒注册插件命令**、对 `gateway` 命令走专门的 fast path（`tryRunGatewayRunFastPath()`）。
- `openclaw.mjs`：npm bin 入口。

### 4.2 Gateway（`src/gateway/`）

- `server.impl.ts` + `server/ws-connection/message-handler.ts`（约 80KB）：WebSocket 服务器、帧校验、RPC 分发、设备配对与认证握手。
- `server-methods/`：方法实现——`sessions.ts`（会话 CRUD/压缩/重置/发送）、`tools-invoke.ts`（工具执行与流式输出）、`models.ts`、`devices.ts`、`environments.ts`。
- `channel-bridge.ts`：协议与通道插件之间的桥接。
- `mcp-http.ts`：网关自身作为 MCP HTTP 服务端的实现。
- 还包含 cron 调度（`server-cron-lazy.ts`）、配置 diff/reload、控制台 UI 路由、健康检查/就绪探针。

### 4.3 Agent 运行时（`src/agents/`）

这是代码量最大的子系统：

- `agent-command.ts`（80KB+）：Agent 命令主调度——会话、运行、模型、投递、重试。
- `command/`：会话解析（`session.ts`）、运行上下文（`run-context.ts`）、尝试执行与 transcript 构建。
- 工具集成：`agent-bundle-mcp-runtime.ts`（38KB，MCP stdio/HTTP 传输生命周期）、`bash-tools.exec-runtime.ts`（35KB，带 PTY 回退与审批流的 Bash 执行）、`agent-bundle-lsp-runtime.ts`（LSP 集成）。
- 模型管理：`model-selection.ts`（模型目录构建/默认解析）、`model-fallback.ts`（错误时模型回退）、`agent-model-discovery.ts`（可用性与认证探测）。
- `agent-run-terminal-outcome.ts`：运行终态的规范化/合并——超时/取消优先级在此单点裁决，禁止下游重新推导（AGENTS.md 明文规定）。
- `harness/` + `embedded-agent-runner/`：可插拔的 agent harness（含进程内运行模式），支持外部编码代理（Codex 等）作为 harness。

### 4.4 通道架构（`src/channels/`）

- `registry.ts` / `bundled-channel-catalog-read.ts`：通道注册表与元数据目录。
- `streaming.ts`（35KB）+ `draft-stream-loop.ts`：**草稿流式输出状态机**——将 LLM 流式 token 聚合为通道可接受的草稿编辑（遵守 `docs/concepts/streaming.md` 的"禁止 token-delta 外发"约定）。
- `conversation-resolution.ts`（16KB）/ `thread-bindings-policy.ts`：消息到会话/线程的映射与绑定策略。
- `inbound-event/` + `inbound-debounce-policy.ts`：入站事件去抖——把短时间内多条消息合并为一次 agent 运行。
- 门控体系：`mention-gating.ts`（@提及）、`command-gating.ts`（命令开关）、`allowlists/`（白名单）、`model-overrides.ts`（按通道模型覆盖）。

### 4.5 消息路由与自动回复（`src/routing/`、`src/auto-reply/`）

完整入站链路：WebSocket/通道事件 → `message-handler.ts` 分发 → `routing/session-key.ts` 构造稳定会话键（`agent:<agent-id>:<main-or-explicit>:<request-key>`）→ `auto-reply/dispatch.ts`（22KB）编排回复（hook 组合、**前台投递栅栏**——新回复抑制过期可见投递）→ 模型选择 → agent 执行 → `auto-reply/reply/route-reply.ts` 投递回原通道。

### 4.6 配置系统（`src/config/`）

体量惊人，体现"配置面即产品面"的设计：

- `io.ts`（99KB+）/ `io.write-prepare.ts` / `io.audit.ts`：`openclaw.json` 的读写、原子写准备、变更审计。
- `schema.ts` + `validation.ts`（64KB+）+ 各子系统 `zod-schema.*.ts`：Zod 校验体系。
- `schema.help.ts`（203KB）/ `schema.labels.ts`（66KB）：每个配置字段的帮助文本与 UI 标签——配置即文档。
- 特性：include 解析、`$ENV` 环境变量替换、`future-version-guard.ts`（拒绝未来版本键）。
- 迁移哲学：运行时只认最新 schema；旧形态由 `openclaw doctor --fix` 检测、解释、备份并重写为规范格式。核心配置由核心 doctor 修复，插件配置由插件自己的 doctor 契约（`legacyConfigRules`/`normalizeCompatibilityConfig`）修复。

### 4.7 状态存储（`src/state/`）

- 驱动：Node 内置 `node:sqlite`（`DatabaseSync`），ORM 为 **Kysely**（类型从 `openclaw-state-db.generated.d.ts` 自 schema 生成）。
- 双库设计：
  - 共享库 `state/openclaw.sqlite`（`openclaw-state-db.ts`，42KB）——全局运行时状态 + 插件 KV；
  - 每 agent 库 `agents/<agentId>/agent/openclaw-agent.sqlite`（`openclaw-agent-db.ts`）——会话 transcript、压缩检查点。
- 含 WAL 维护、0o600 权限、备份/迁移审计表（`backup_runs`、`migration_runs`）、查询计划分析（`sqlite-query-plan.ts`）。
- 策略：原始 SQL 仅限 DDL/迁移/底层引导；遗留文件存储一律视为迁移债务。

### 4.8 LLM/Provider 运行时（`src/llm/`、`src/model-catalog/`、`src/provider-runtime/`）

- `llm/api-registry.ts` / `model-registry.ts` / `env-api-keys.ts`：Provider API 注册、模型注册、环境密钥加载。
- `model-catalog/`：模型权威目录与 manifest 规划（manifest 与运行时双 authority）。
- `provider-runtime/operation-retry.ts`：供应商调用重试。
- 实际 Provider 实现全部下沉到插件（`extensions/anthropic`、`extensions/openai` 等），核心只持有通用循环——"Providers own auth/catalog/runtime hooks; core owns generic loop"。

## 5. 插件体系（`extensions/` + Plugin SDK）

### 5.1 Plugin SDK（`packages/plugin-sdk/`、`src/plugin-sdk/`）

- **API barrel 设计**：SDK 拆分为 50+ 个轻量子路径（`./plugin-runtime`、`./channel-runtime`、`./provider-auth-runtime`、`./tts-runtime`、`./exec-approvals-runtime` 等），插件按需导入，避免巨型 barrel 拖慢启动。
- 入口工厂：`definePluginEntry()`（`src/plugin-sdk/plugin-entry.ts`），暴露 `OpenClawPluginApi`/`OpenClawPluginDefinition`。
- **Manifest 优先**：规范文件 `openclaw.plugin.json`（上限 256KB，LRU 缓存），声明 `id`、`kind`（provider/channel/tool/hook）、`configSchema`（带 UI 提示的 JSON Schema）、激活条件（`onStartup`/`onProviders`/`onChannels`/`onCommands`/`onCapabilities`/`onConfigPaths`）、模型目录/定价/前缀匹配、secret 集成、兼容性迁移路径等。控制平面（发现/校验/setup）完全基于元数据工作，无需加载插件运行时。

### 5.2 插件加载器（`src/plugins/`）

- `discovery.ts`：扫描内置（`extensions/`）与外部插件目录，产出 `PluginCandidate`，512 条目 LRU + mtime 校验缓存 manifest。
- `activation-planner.ts`：基于 manifest 激活元数据规划加载哪些插件——配置了哪个通道/供应商/命令才加载对应插件。
- `loader.ts`：发现 → 校验 → manifest 加载 → API 构建 → 注册的总编排。
- 内置 vs 外部：内置插件随核心 dist 发布；外部官方插件自有 package/依赖，通过 `@openclaw/plugin-package-contract` 校验 `openclaw.compat.pluginApi` 兼容性元数据。

### 5.3 143 个插件的分类版图

| 类别 | 数量级 | 代表 |
|---|---|---|
| 消息通道 | ~24 | telegram、slack、discord、whatsapp、signal、matrix、imessage、msteams、feishu、line、qqbot、zalo |
| LLM 供应商 | ~35 | anthropic、openai、google、amazon-bedrock、mistral、groq、deepseek、qwen、moonshot、ollama、llama-cpp、vllm、openrouter |
| 语音/电话 | ~8 | elevenlabs、deepgram、azure-speech、talk-voice、voice-call、phone-control |
| 图像/视频/媒体 | ~8 | image-generation-core、comfy、fal、runway、pixverse、video-generation-core |
| 记忆/搜索 | ~9 | memory-core、memory-lancedb、memory-wiki、brave、exa、tavily、searxng |
| 编码代理/工具 | ~13 | codex、opencode、openshell、browser、firecrawl、document-extract |
| 基础设施/其他 | 其余 | diagnostics-otel、diagnostics-prometheus、admin-http-rpc、canvas、device-pair、policy、migrate-claude |

典型插件结构：`package.json`（含 `"openclaw"` 块）+ `openclaw.plugin.json`（manifest）+ `index.ts`（入口）+ `api.ts`（公共契约）+ `runtime-api.ts`（懒加载运行时）+ `src/`（私有实现，禁止被外部深引）。

## 6. 共享包（`packages/`，21 个）

按依赖层次：

- **协议层**：`gateway-protocol`（仅依赖 TypeBox）← `gateway-client` ← `sdk`（高层封装）。
- **LLM 层**：`llm-core`（消息类型/补全 schema/事件流）← `llm-runtime`、`agent-core`（AgentHarness、会话存储、消息校验、分支摘要）；`model-catalog-core`（模型引用/定价规范化）；`tool-call-repair`（LLM 工具调用 schema 修复）。
- **媒体层**：`media-core`、`media-generation-core`、`media-understanding-common`、`speech-core`。
- **领域工具**：`memory-host-sdk`（记忆引擎接口）、`web-content-core`、`markdown-core`、`terminal-core`、`normalization-core`、`net-policy`（网络策略）、`acp-core`（Agent Client Protocol）。
- **插件契约**：`plugin-sdk`（公共 SDK barrel）、`plugin-package-contract`（外部插件兼容性校验）。

## 7. 网关协议（`packages/gateway-protocol/`）

- 当前协议版本 **v4**（`packages/gateway-protocol/src/version.ts:2`，`PROTOCOL_VERSION = 4`，最低客户端版本同为 4）。
- TypeBox schema 定义全部消息类型，编译期校验器（Compile API）。
- 覆盖面：Agents API（身份/CRUD/文件/事件）、Channels API（启停/登出）、Talk Sessions（音频/文本对话、工具调用、转向、模型选择）、Artifacts（产物下载）、Runners。
- 治理规则：协议变更必须先做加法；不兼容变更需要版本化 + 文档 + 客户端跟进；版本号提升仅限 owner 显式确认。

## 8. 工程基础设施

### 8.1 构建

- 构建器：**tsdown**（Rolldown 后端），输出 `dist/`；TypeScript 严格模式，ES2023 + NodeNext。
- 多 tsconfig 工程拆分：`tsconfig.core.json`、`tsconfig.extensions.json`、`tsconfig.plugin-sdk.dts.json` 等。
- 类型检查走 **tsgo** lane（明确禁止新增 `tsc --noEmit`）。
- Node 要求 `>=22.19.0`（推荐 Node 24），同时保持 Bun 路径可用；npm 发布带 `npm-shrinkwrap.json` 锁定。
- pnpm 配置含 125+ patch 排除与 ~53 个依赖 override 钉版本，且对供应链有最小发布时长（2880 分钟）防护。

### 8.2 测试

- **Vitest**，6,265 个测试文件，112 个按域分片的配置（`test/vitest/`）：unit/infra/boundary/contract（通道契约、插件契约）/extension 分片/e2e/UI-e2e/gateway 各子域。
- 测试就近放置（`*.test.ts` 与源码同目录），e2e 用 `*.e2e.test.ts`，live 测试用 `OPENCLAW_LIVE_TEST=1` 门控。
- `scripts/run-vitest.mjs` 提供进程组清理、看门狗超时；远程/全量验证走 Crabbox/Testbox 基础设施。
- 契约测试是亮点：通道契约、插件契约作为独立分片，保护 SDK 边界。

### 8.3 Lint/格式化

- 全 Rust 工具链：**oxlint**（correctness/perf/suspicious 全部 error 级，50+ 规则）+ **oxfmt**（行宽 100），不用 ESLint/Prettier。
- 架构守护：import-cycle 检查（`pnpm check:import-cycles`）、madge、包边界 tsconfig（`extensions/tsconfig.package-boundary.*`）、`security/opengrep/` 语义 grep 规则。

### 8.4 CI/CD（59 个工作流）

- `ci.yml`：preflight 先裁决跑什么（docs_only/run_node/run_macos/run_android…），再多分片测试矩阵 + lint/format/boundary/契约检查，覆盖 Node/macOS(Swift)/Android(Kotlin)/Windows。
- 发布链：`openclaw-npm-release.yml`（tag 触发、构件 attestation/provenance）、`docker-release.yml`（多阶段构建、SHA256 钉死基础镜像、ghcr.io）、`plugin-npm-release.yml`/`plugin-clawhub-release.yml`（插件双注册表发布，使用 trusted publishing）。
- 质量护栏：CodeQL（含 Android/macOS 专项）、`dependency-guard.yml`（供应链）、live 通道冒烟（mantis-discord/telegram/slack）、跨 OS 安装冒烟、性能基线（`openclaw-performance.yml`）。
- 版本方案：`YYYY.M.PATCH` 月度发布列车（PATCH 为序号而非日期），beta/alpha 预发布有严格的列车推进规则。

## 9. 客户端：原生应用与 UI

- `apps/macos`、`apps/ios`：Swift（SwiftUI，规范要求用 Observation 框架）；`apps/macos-mlx-tts`：基于 MLX 的本地 TTS；`apps/swabble`：独立 Swift 包。
- `apps/android`：Kotlin/Gradle。
- `apps/shared`：跨平台共享代码。
- `ui/`：**Lit 3 Web Components** 控制台（`openclaw-control-ui`），Vite 8 构建，Vitest + Playwright 浏览器测试；依赖 markdown-it/DOMPurify/highlight.js。
- 所有客户端经由 gateway-protocol v4 与网关通信。

## 10. 部署

- **Docker 为主**：多阶段 Dockerfile（bookworm → bookworm-slim），Node 24 + Bun，`OPENCLAW_EXTENSIONS` build-arg 可选打包插件；docker-compose 支持挂载配置/工作区/凭据，可选 Docker socket 实现沙箱。
- PaaS：Fly.io（`fly.toml`，/data 持久卷 + `/health` 健康检查）、Render（`render.yaml`）。
- 另有 Nix 安装路径（独立仓库 `openclaw/nix-openclaw`）与安装脚本（sibling 仓库 `openclaw.ai`）。

## 11. 安全模型

- **威胁模型**：本地优先、可信操作员（trusted-operator），非多租户 SaaS；无边界绕过的纯 prompt-injection 不在响应范围（`SECURITY.md`）。
- 密钥约定：通道/供应商凭据在 `~/.openclaw/credentials/`，模型认证在 per-agent `auth-profiles.json`；状态库文件 0o600；禁止提交真实凭据。
- 防护设施：`security/opengrep/` 规则、pre-commit hooks、secret scanning 流程、GHSA 流程、锁文件被明确列为安全面（要求 review）。
- 网络策略包 `packages/net-policy` + SSRF 防护 runtime（plugin-sdk 中的 `ssrf-runtime-internal`）。

## 12. 关键设计模式与技术亮点

1. **Manifest 优先的两平面插件架构**——控制平面（发现/校验/激活规划）只读元数据，运行平面按需懒加载；143 个插件不拖累启动。
2. **激进的懒加载纪律**——CLI fast path、`*.runtime.ts` 懒边界（构建期有 `[INEFFECTIVE_DYNAMIC_IMPORT]` 检查防止静态+动态混用同一模块）、SDK 50+ 细分子路径。
3. **"医生模式"迁移哲学**——运行时零兼容垫片，所有旧形态由 `openclaw doctor --fix` 单点迁移；兼容性是显式 opt-in 的产品决策而非实现便利。这在大型长寿项目中相当少见且自律。
4. **数据库优先状态管理**——共享库 + per-agent 库的 SQLite 双层设计，Kysely 类型安全，文件存储必须是命名产品产物（导入/导出/日志/备份）。
5. **热路径事实前移**——provider id、model ref、channel id 等"prepared facts"随请求携带，禁止请求期重复发现与新鲜度轮询（no `stat`/reread on hot paths）。
6. **契约测试 + 边界检查**作为架构防腐层：包边界 tsconfig、import-cycle 门禁、通道/插件契约分片。
7. **全 Rust 前端工具链**（oxlint/oxfmt/tsgo/tsdown-Rolldown），在 1.6 万 TS 文件规模下保证检查速度。
8. **AGENTS.md 作为可执行治理**——根与各子树的 AGENTS.md 把架构规则写成硬策略（LOC 预算、回退审批、配置面新增门槛），并被 AI 审查器（ClawSweeper）消费，是"AI 原生工程治理"的样本。

## 13. 风险与改进观察

1. **核心体量与单文件巨石**：`src/config/io.ts`（99KB）、`agents/agent-command.ts`（80KB+）、`gateway/.../message-handler.ts`（80KB）远超仓库自己 ~700 行拆分建议，是维护热点与合并冲突高发区。
2. **配置面规模**：`schema.help.ts` 203KB 帮助文本侧面反映 `openclaw.json` 配置面已非常庞大——仓库自己也承认（"Config/env surface bar is high"），新增配置项门槛规则正是对此的对冲。
3. **插件数量 vs 维护带宽**：143 个插件（其中 ~35 个 LLM 供应商）大多依赖外部 API，契约漂移风险高；live 测试矩阵只覆盖头部通道（Discord/Telegram/Slack），长尾插件实际回归依赖社区反馈。
4. **444 条 npm scripts** 与 59 个 CI 工作流：工程自动化丰富但入口繁多，新贡献者认知成本高（AGENTS.md/CONTRIBUTING 在一定程度上缓解）。
5. **协议 v4 最低版本即当前版本**：客户端兼容窗口为零，意味着网关与原生应用必须同步升级——对自托管单用户场景可接受，但分发渠道（App Store 审核延迟）可能造成短暂不可用。
6. **供应链面**：55 个 root 运行时依赖 + 大量插件局部依赖 + patch/override 体系，虽有 dependency-guard、最小发布时长、shrinkwrap 等防护，仍是该规模产品的主要攻击面之一。

## 14. 总结

OpenClaw 是一个工程成熟度很高的大型 TypeScript monorepo：**以 WebSocket 网关（协议 v4）为控制平面，以 manifest 驱动的 143 插件生态承载通道与模型供应商，以 SQLite+Kysely 承载状态，以 Agent 运行时（工具/MCP/Bash/模型回退）承载核心智能**。其最显著的特征不是某项单点技术，而是把架构纪律（单一规范路径、doctor 迁移、传输层纯粹性、懒加载边界、LOC 预算）写成机器可执行的治理规则，并配以 6,000+ 测试与 59 条 CI 流水线来强制执行——这使得它在 40 万行级别的代码量与高速迭代（月度发布列车）下仍保持了清晰的所有权边界。
