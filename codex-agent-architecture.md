# Codex 智能体能力与内核架构笔记

> 来源：直接阅读本仓库（`D:\Work\03github\codex`）源码整理，非官方文档。
> 覆盖范围：LLM 相关能力（RAG / Memory / Skill / Plugin / MCP）、contributor 扩展机制、
> 智能体内核模块地图，以及 turn / thread / session 的边界与 resume 行为。
> 不含 sandbox、SDK、app-server 协议细节。

---

## 目录

1. [总体架构：三层](#1-总体架构三层)
2. [能力清单](#2-能力清单)
3. [RAG 的真实情况](#3-rag-的真实情况)
4. [Memory 实现](#4-memory-实现)
5. [Skill 实现](#5-skill-实现)
6. [Plugin 实现](#6-plugin-实现)
7. [MCP 实现](#7-mcp-实现)
8. [其他 LLM / Agent 能力](#8-其他-llm--agent-能力)
9. [Contributor 是什么](#9-contributor-是什么)
10. [核心智能体模块地图](#10-核心智能体模块地图)
11. [turn / thread / session 的边界](#11-turn--thread--session-的边界)
12. [退出、clear、resume、fork](#12-退出clearresumefork)
13. [关键文件索引](#13-关键文件索引)

---

## 1. 总体架构：三层

| 层 | 位置 | 作用 |
|---|---|---|
| Extension 框架 | `codex-rs/ext/extension-api/src/lib.rs` | 每个能力注册成 contributor（Context / Tool / Turn / Lifecycle / Config / ModelRequest…），由 `registry.rs` 统一装进会话 |
| 上下文层 | `codex-rs/core/src/context/mod.rs`、`core/src/context/world_state/mod.rs` | 所有注入模型的内容都是 `ContextualUserFragment`，有类型、有 role、有大小上限 |
| 工具层 | `codex-rs/tools/src/lib.rs`、`codex-rs/core/src/tools/mod.rs` | 统一 `ToolExecutor` + `ToolSpec`；本地工具、MCP 工具、code-mode 工具走同一套 |

`codex-rs/ext/extension-api/notes.md` 里有一张作者自己写的设计表，说明每类能力该用哪类 contributor：

```text
git attribution              Context
memories                     Context + Tool + Output
guardian                     Context + Request
goal                         Tool + Runtime
image generation             Tool + Output
skills                       Context + Turn
personality                  Context
plugins / apps / connectors  Context + Turn
shell snapshot               Runtime
web search                   Tool
AGENTS.md                    Context   (Runtime too only if you want eager refresh/cache behavior)
future sandboxing            probably Request + Runtime
```

---

## 2. 能力清单

| 能力 | 是否涉及 | 实现要点 |
|---|---|---|
| **RAG** | ❌ 无向量检索 | 没有 embedding / 向量库 / 语义索引。全仓库搜 `embedding` 只命中注释。替代品见第 3 节 |
| **Memory** | ✅ 完整 | 两阶段 LLM 抽取管线 + 文件存储 + 检索工具 |
| **Skill** | ✅ 完整 | `SKILL.md` 解析、目录扫描、catalog 注入、显式/隐式调用，外加一套 BM25 词法选择器 |
| **Plugin** | ✅ 完整 | 声明式资源包（可含 skills / MCP / apps / hooks），marketplace 安装，进程外执行 |
| **MCP** | ✅ 完整 | 完整 MCP 客户端栈，含 OAuth、elicitation、resources、工具延迟加载 |

---

## 3. RAG 的真实情况

**结论：Codex 没有向量 RAG。** 搜遍 `codex-rs`，`embedding` 只出现在注释里（如 app-server README 的 "embedding host"），没有任何 embedding 模型调用、向量库或语义索引。

它用"检索"替代，但全部是词法/结构化检索：

1. **技能检索**（最接近 RAG 的东西）—— `codex-rs/ext/skills/src/dynamic_skill_selector.rs` 及其子模块：
   - `fielded_bm25.rs` — 分字段 BM25
   - `character_ngram.rs` — 字符 n-gram
   - `multi_query_lexical.rs` — 多查询词法
   - `rrf_lexical_char.rs` — RRF（Reciprocal Rank Fusion）融合
   - `lru_plus_lexical.rs` / `lru_plus_character_routing.rs` — LRU 与词法/字符路由混合
   - 统一 `CheapSkillSelector` trait，契约要求"确定性、无副作用、每轮都能跑"
   - **注意**：当前是影子实验（`Feature::SkillSearch`），只记录指标，不改模型可见的 catalog
2. **记忆检索** — `ext/memories/src/local/search.rs`：纯子串 / 逐行匹配，支持 `Any` / `AllOnSameLine` / `AllWithinLines{n}` 三种模式，上限 200 条，带 cursor 分页。**没有任何打分排序**。
3. **工具检索** — `codex-rs/tools/src/tool_search.rs` + `defer_loading`：MCP 工具默认不进上下文，靠 `tool_search` 按需拉取。
4. **文件搜索** — `file-search` crate 的模糊文件名匹配。
5. **网页搜索** — `ext/web-search/src/tool.rs` 的 `web.run` 工具，实际调用后端 `codex_api::SearchClient`（`codex-rs/codex-api/src/endpoint/search.rs`），检索在服务端，本地不做索引。

---

## 4. Memory 实现

分 read / write 两条独立路径，互不依赖（`memories/read/src/lib.rs` 特意注明 "intentionally does not depend on the memory write pipeline"）。

### 写入（`codex-rs/memories/write/`）

后台启动任务（`start_memories_startup_task`），两阶段：

**Phase 1（`phase1.rs`）** — 从 rollout 抽取：

- `reasoning_effort = Low`
- 并发 8（`CONCURRENCY_LIMIT`）
- 最多扫 5000 个 thread（`THREAD_SCAN_LIMIT`）
- rollout 输入只占模型上下文窗口的 70%（`CONTEXT_WINDOW_PERCENT`），fallback 截断 150k token

**Phase 2（`phase2.rs`）** — 合并固化：

- `reasoning_effort = Medium`
- 先算一份 workspace diff（`phase2_workspace_diff.md`，≤4MB）再让模型改
- 心跳 90s，租约 3600s

**产物布局**（落在 `CODEX_HOME/memories/`）：

```text
memories/
  raw_memories.md          # 抽取出的原始记忆
  rollout_summaries/       # 每个 rollout 的摘要
  extensions/              # 额外记忆源，含 instructions.md
    ad_hoc/notes/          # 用户在对话中提出"记住/忘掉"时追加的小笔记
```

其他保障：

- `guard.rs` 做安全校验
- `ensure_layout()` 拒绝 memory root 是符号链接，并清理目录内的 symlink
- extension 资源保留 7 天（`RETENTION_DAYS`），prune 批量 200

### 读取（`codex-rs/ext/memories/src/extension.rs`）

同时做两件事：

**1. `ContextContributor`** — 注入 memory summary

- 作为 developer policy 注入（`PromptFragment::developer_policy`）
- V2 版本按 8900 字节切片，保证每片不超限
- 对应 `core/src/context/memory.rs` 的 `MemoryContextFragment::ReadInstructions`，`TruncationPolicy::Bytes(8_900)`

**2. `ToolContributor`** — 注册 `memories` namespace 下的四个工具

| 工具 | 说明 |
|---|---|
| `add_ad_hoc_note` | 追加用户显式的记住/忘掉请求 |
| `list` | 默认上限 2000 |
| `read` | 默认 20000 token |
| `search` | 默认 200，最多 200 |

### 引用溯源与埋点

- `memories/read/src/citations.rs` — 模型回答里要输出 `<oai-mem-citation>` 块，`memory_summary.md` 不可引用
- `memories/read/src/usage.rs` — 解析 shell 命令，判断模型实际读了哪个 memory 文件，按 `MemoryVersion`（V1/V2）和 `MemoriesUsageKind`（MemoryMd / MemorySummary / RawMemories / RolloutSummaries / Skills）分类上报
- 提示词明确：memory 不是当前行为的证明；不许直接改生成文件，只能写 ad-hoc 笔记，由 consolidation 处理

### 相关 feature 开关

| Feature | 含义 |
|---|---|
| `MemoryTool` | 启用启动期记忆抽取与文件固化 |
| `ExternalAgentMemoryImport` | 从其他 agent 导入项目记忆 |
| `Chronicle` | 实验性"被动屏幕上下文记忆" sidecar |

---

## 5. Skill 实现

### 核心 crate：`codex-rs/skills/`

| 模块 | 职责 |
|---|---|
| `parser.rs` | 解析 `SKILL.md` frontmatter |
| `loading.rs` | 目录扫描、快照缓存 |
| `interface.rs` | 解析 `agents/openai.yaml` |
| `model.rs` | `SkillMetadata` / `SkillDependencies` / `SkillPolicy` |
| `mentions.rs` | `$SkillName` / `skill://` 提及解析 |

### 内置技能编译进二进制

```rust
const SYSTEM_SKILLS_DIR: Dir = include_dir::include_dir!("$CARGO_MANIFEST_DIR/src/assets/samples");
```

启动时释放到 `CODEX_HOME/skills/.system`，靠一个含指纹的 marker 文件（`.codex-system-skills.marker`）避免重复写入。

当前内置技能：`imagegen`、`openai-docs`、`review-agent`、`skill-creator`、`skill-installer`。

### 运行时扩展：`codex-rs/ext/skills/`

| 模块 | 职责 |
|---|---|
| `catalog.rs` | 技能目录（`SkillCatalog` / `SkillAuthority` / `SkillPackageId`） |
| `selection.rs` | 显式选择：`skill://` 路径、`$name` 提及、`UserInput::Skill` |
| `invocation.rs` | 隐式调用检测 |
| `render.rs` | 渲染成上下文片段 |
| `dynamic_skill_selector/` | 词法选择器（影子实验） |
| `world_state.rs` | `host_skills` world state section |
| `sources.rs` / `provider/` | Host 与 Executor 两种技能来源 |

### 技能形态

```text
<skill-name>/
  SKILL.md          # 提示词正文 + frontmatter
  references/       # 按需读取的参考文档
  scripts/          # 可执行脚本
  assets/           # 图片等资源
  agents/openai.yaml
```

本质是"提示词 + 资源"，不是代码插件。

---

## 6. Plugin 实现

**关键点：插件不是动态链接库，是可安装的声明式资源包。**

### Manifest（`codex-rs/plugin/src/manifest.rs`）

```rust
pub struct PluginManifestPaths<Resource> {
    pub skills: Vec<Resource>,
    pub onboarding_skill: Option<Resource>,
    pub mcp_servers: Option<PluginManifestMcpServers<Resource>>,
    pub apps: Option<Resource>,
    pub hooks: Option<PluginManifestHooks<Resource>>,
}
```

外加 `interface`（`display_name`、`logo`、`screenshots`、`capabilities` 等给 UI 用的元数据）。

### 生态管理（`codex-rs/core-plugins/`）

- marketplace 常量：`openai-curated`、`openai-api-curated`、`openai-bundled`、`openai-primary-runtime`
- 安装 / 卸载 / 升级（`manager.rs`、`marketplace_*.rs`）
- remote plugin（`remote*.rs`）
- 推荐插件与 `request_plugin_install` 审批流

### 插件带来的可执行能力只有两条路

1. **Hooks** — `plugin/src/bundled_hooks.rs`（有 allowlist 检查 `is_allowlisted_bundled_cleanup_hook`）
2. **MCP servers** — `ext/mcp/src/plugin/`（`discovery.rs`、`provider.rs`）

**没有任意 Rust 代码注入点。**

---

## 7. MCP 实现

### 主入口：`codex-rs/codex-mcp/`

| 模块 | 职责 |
|---|---|
| `catalog.rs` | 服务器注册、冲突消解（`McpServerConflictAction`） |
| `connection_manager.rs` | 连接生命周期（**AGENTS.md 要求改 MCP 工具调用优先动这里**） |
| `binding.rs` | 工具调用绑定（`McpBinding` / `PreparedMcpCall`） |
| `tools.rs` | `ToolInfo` |
| `rmcp_client.rs` | 底层协议客户端 |
| `runtime.rs` | `McpRuntime` / `McpStartupPolicy` |
| `resource_client.rs` | MCP resources 的列表/读取/分页与事件流 |

### 认证与授权

- `mcp/auth.rs`、`trusted_access.rs`
- `auth_elicitation.rs` — 工具调用缺授权时弹 elicitation
- `ema_auth.rs`
- Feature 开关：`McpOAuthRefreshCoordination`、`UseXaa`

### 协议版本与工具暴露

- `Mcp20260728`、`CodexAppsMcp20260728` — 协议版本开关
- MCP 工具默认 `defer_loading`，只在 `tool_search` 命中后才进上下文
- `NonPrefixedMcpToolNames` — 是否去掉 `mcp__` 前缀
- `CODEX_APPS_MCP_SERVER_NAME` — host 自有的 Codex Apps server

### 扩展接入：`codex-rs/ext/mcp/`

`plugin_contributor.rs`、`plugin_providers.rs`、`stream_manager.rs`、`cloud_plugin.rs`。

---

## 8. 其他 LLM / Agent 能力

| 能力 | 位置 | 要点 |
|---|---|---|
| **上下文与 world state** | `core/src/context/world_state/mod.rs` | 模型可见状态切成带稳定 ID 的 section，每轮只渲染相对上一份 snapshot 的 RFC 7386 merge patch，SHA1 指纹防抖 |
| **Compaction** | `core/src/compact.rs`、`compact_remote_v2*.rs` | 区分 pre-turn / mid-turn 压缩，配 `PreCompact` / `PostCompact` hook，含 model fallback 与 token budget |
| **Code Mode** | `codex-rs/code-mode-runtime/`（`v8_init.rs`） | 内嵌 V8，把工具定义转成 JS 定义，模型可写 JS 调工具；`CodeModeOnly` 下只剩 `exec` / `wait` |
| **多智能体** | `core/src/agent/`、`ext/agent`、`agent-roles`、`agent-graph-store` | 工具：`spawn_agent` / `send_input` / `wait` / `close_agent` / `resume_agent`；`MultiAgentV2` 启用 task-path 路由 |
| **Guardian** | `guardian-context/`、`core/src/guardian/`、`ext/guardian-v2` | 可组合的审查 context section，分同步审查与异步打分，严格 token 预算 |
| **Hooks** | `codex-rs/hooks/` | 12 个事件，见下 |
| **Apps / Connectors** | `codex-rs/connectors/` | 从 ChatGPT 后端拉应用目录，1 小时磁盘+内存缓存，通过 Codex Apps MCP server 变成工具 |
| **模型层** | `codex-rs/codex-api/src/endpoint/`、`model-provider/` | Responses API（HTTP + WebSocket）、Realtime call、search、images、memories；Bedrock / Ollama / LM Studio |
| **Realtime / 语音** | `realtime-webrtc/`、`voice-host/`、`core/src/realtime_*.rs` | WebRTC 实时会话 |
| **图像生成** | `ext/image-generation/` | namespace `image_gen`，工具名 `imagegen` |
| **其他扩展** | `ext/queue`、`ext/history-notes`、`ext/git-attribution`、`ext/goal`、`ext/items` | 消息排队、历史笔记、Git 归因、目标/预算、类型化展示项 |

### Hook 事件全表（`codex-rs/hooks/src/lib.rs`）

```text
PreToolUse, PermissionRequest, PostToolUse,
PreCompact, PostCompact,
SessionStart, SessionEnd, UserPromptSubmit,
SubagentStart, SubagentStop,
Stop, Interrupt
```

执行方式只有两种：shell 命令（`command_runner.rs`）和 MCP（`mcp_runner.rs`）。

### 设计取向

Codex 没做向量 RAG，而是用三个更确定的东西替代：

1. **结构化的上下文注入** — `ContextualUserFragment` + world state merge patch，保证有界、增量、缓存友好
2. **LLM 驱动的离线记忆固化** — rollout → Phase1 抽取 → Phase2 合并
3. **轻量词法检索** — BM25 / n-gram / tool_search，全部本地确定性计算

外部数据统一走 MCP / connectors，而不是塞进本地索引。

---

## 9. Contributor 是什么

**一句话：contributor 是 core 给扩展留的"回调插槽"，本质是一组 trait 对象。**

它解决的具体矛盾：Codex 想让 memory、skills、MCP、goal 深度参与智能体行为，但 AGENTS.md 又明确写着 **"resist adding code to codex-core"**。contributor 就是解法——**依赖倒置**。

core 完全不认识 memory、skill、MCP，只认识 `ext/extension-api/src/contributors.rs` 里的 trait。

### Trait 全表

| Trait | 被调用的时机 | 典型实现者 |
|---|---|---|
| `ContextContributor` | 组装模型输入时 | memories（注入摘要）、git-attribution、skills |
| `ToolContributor` | 装配本轮工具表时 | memories（4 个工具）、image-generation、goal |
| `TurnInputContributor` | 每次提交 turn 时补输入片段 | 队列、hook 附加上下文 |
| `McpServerContributor` | 解析运行时 MCP server 时 | mcp 扩展、plugins |
| `ThreadLifecycleContributor` | thread start / ready / resume / idle / stop | memories（初始化）、queue |
| `TurnLifecycleContributor` | turn start / item completed / stop / abort / error | guardian、goal |
| `ToolLifecycleContributor` | tool dispatch / start / command start / mcp result / finish / timing | 审计、归因 |
| `ApprovalReviewContributor` | 审批决策时（可"认领"请求） | guardian-v2 |
| `ModelRequestContributor` | 模型响应拦截 | guardian 评分 |
| `TurnItemContributor` | turn item 发出前可变后处理 | 展示层加工 |
| `SkillInvocationContributor` | 技能被调用后 | 技能遥测 |
| `TokenUsageContributor` | 每次记录 token 用量后 | 预算类能力 |
| `ConfigContributor` | 配置变更提交后 | memories（版本切换） |

另有 registry 级的两项：`TurnStartAdmission`（turn 准入门）和 `ExtensionEventSink`（事件出口）。

### 三个关键机制

**1. 注册 → 冻结 → 遍历**

```text
ExtensionRegistryBuilder  →  收集 Arc<dyn XxxContributor>
        build()
ExtensionRegistry         →  不可变
        core 在固定时机遍历调用
```

注册表里每个类别都是 `Vec<Arc<dyn Trait>>`。两种消费语义：

- **广播**：所有 contributor 都跑（如 context）
- **认领**：第一个返回 `Some` 的赢（如 `decide_approval`）

**2. 三层 store 做状态隔离**

contributor 拿不到 core 的运行时对象，只能通过 `ExtensionData` 读写自己的私有状态：

| Store | 作用域 | 生命周期 |
|---|---|---|
| `session_store` | 整个会话 | 进程 |
| `thread_store` | 单条 thread | thread（软重启后重建） |
| `turn_store` | 单个 turn | turn |

契约上要求 contributor **不能持有会话状态**，只能借用当次输入——所以同一个 registry 能被并发复用。

**3. 唯一的装配点**

哪个 session 装了哪些能力，只有一处真相：`codex-rs/app-server/src/extensions.rs` 的 `thread_extensions()`。

```rust
codex_queue_extension::install(&mut builder, queue_service);
codex_history_notes_extension::install(&mut builder, auth_manager.clone());
codex_core::install_agent_message_board(&mut builder, thread_manager.clone());
codex_goal_extension::install_with_backend(...);
codex_git_attribution::install(...);
codex_guardian_v2::install(&mut builder, auth_manager.clone(), thread_manager);
codex_memories_extension::install(&mut builder, codex_otel::global());
codex_mcp_extension::install(&mut builder);
codex_mcp_extension::install_plugins(&mut builder, environment_manager);
codex_web_search_extension::install(&mut builder, auth_manager.clone());
codex_image_generation_extension::install(&mut builder, ...);
codex_skills_extension::install_with_providers_and_metrics(...);
```

core 侧的消费方式（`core/src/session/world_state.rs`）：

```rust
for contributor in self.services.extensions.context_contributors() {
    for section in contributor
        .contribute_world_state(WorldStateContributionInput { ... })
        .await
    {
        world_state.add_extension_section(section);
    }
}
```

### 两个容易混淆的点

- **contributor ≠ plugin**：plugin 是**数据**（声明式资源包），contributor 是**编译期代码接口**。plugin 只能通过 `McpServerContributor::selected_plugins()` 间接进来。
- **contributor ≠ hook**：hook 是用户可配置的外部 shell / MCP 命令，**能改 tool payload**；contributor 是进程内 Rust 代码，`ToolLifecycleContributor` 注释明确写 "observe... without rewriting the invocation"。

---

## 10. 核心智能体模块地图

去掉 sandbox / SDK / app-server 协议后，真正的"智能体内核"：

| 层 | 位置 | 职责 |
|---|---|---|
| **① 模型接入** | `codex-api/src/endpoint/`（responses / responses_websocket / realtime_call / images / search / memories）+ `model-provider/` + `core/src/client.rs` | Responses API 调用、鉴权、provider 选择、WS 连接复用与 `previous_response_id` 增量 |
| **② 会话与回合循环** | `core/src/session/`（`session.rs` / `turn.rs` / `step_context.rs` / `turn_context.rs` / `submission.rs` / `input_queue.rs`）+ `core/src/tasks/`（`regular` / `compact` / `review` / `user_shell`） | 一次 turn 的完整状态机；turn 的四种形态 |
| **③ 上下文构造** | `core/src/context/` + `context/world_state/` + `context_manager/` + `token_budget.rs` + `retained_context.rs` | 片段类型化、增量渲染、预算裁剪、压缩后保留 |
| **④ 压缩** | `core/src/compact.rs` + `compact_remote_v2*.rs` + `compact_token_budget.rs` | 上下文超限时的摘要素替代 |
| **⑤ 工具** | `core/src/tools/`（`registry` / `router` / `orchestrator` / `spec_plan` / `parallel` / `lifecycle` / `executed_tool_calls`） | 工具表装配、路由、并发执行、生命周期埋点 |
| **⑥ 扩展** | `codex-rs/ext/` 全部 | 第 9 节那 13 个 trait 的实现方 |
| **⑦ 记忆与持久化** | `memories/read` + `memories/write` + `rollout` + `thread-store` + `history` + `state` | 会话落盘、rollout 元数据入库、记忆抽取与注入 |
| **⑧ 多智能体** | `core/src/agent/`（`control.rs` / `role.rs` / `registry.rs` / `agent_resolver.rs`）+ `thread_manager.rs` + `agent-graph-store` | 角色解析、子 thread 派生、父子拓扑、控制通道 |
| **⑨ 安全审查** | `core/src/guardian/` + `guardian-context/` + `ext/guardian-v2` | 审批决策与异步风险打分 |

### 一次 turn 里 contributor 的调用顺序

```text
用户提交 turn
   │
   ├─ TurnStartAdmission::admit_turn_start()          ← 准入门（registry 级）
   │
   ├─ TurnInputContributor::contribute()              ← 补输入片段
   │
   ├─ TurnLifecycleContributor::on_turn_start()       ← 本轮初始化
   │     （requires_mcp_runtime 决定是否先 reconcile MCP）
   │
   ├─ ContextContributor::contribute_turn_context()   ← 拼 prompt
   ├─ ContextContributor::contribute_world_state()    ← 状态差异章节
   │     └─ WorldState::render_diff() → merge patch → ContextualUserFragment
   │
   ├─ ModelRequestContributor                           ← 模型调用
   │     （沙箱/审批在此请求；响应可被拦截评分）
   │
   ├─ ToolContributor::tools()                          ← 装配工具表
   ├─ ToolLifecycleContributor::on_tool_dispatch/start
   ├─ ApprovalReviewContributor::decide()               ← 需要审批时认领
   ├─ ToolLifecycleContributor::on_tool_finish
   │
   ├─ TurnItemContributor::contribute()                 ← item 发出前后处理
   ├─ TokenUsageContributor::on_token_usage()
   └─ TurnLifecycleContributor::on_turn_stop/abort/error
```

---

## 11. turn / thread / session 的边界

| 概念 | 代码载体 | 生命周期 | 是否持久 |
|---|---|---|---|
| **turn** | `ActiveTurn`（`Session.active_turn: Mutex<Option<ActiveTurn>>`） | 一次用户提交 → 模型跑完所有工具调用 → 产出最终回复 | 否，但写进 rollout |
| **thread** | `ThreadId` + `CodexThread` + 一个 rollout 文件 | 从 `thread/start` 到 `on_thread_stop` | ✅ 唯一持久句柄 |
| **Session**（代码结构体） | `core/src/session/session.rs` 的 `Session` | 与 thread 运行时一一对应，随进程存活 | 否 |

### 关键映射

```text
用户界面上的 "session"  ==  代码里的 thread  ==  一个 rollout 文件
```

`thread-store/src/lib.rs` 的注释：

> Application code should treat `ThreadId` as the only durable thread handle.

`CodexThread` 的注释：

> Conduit for the bidirectional stream of messages that compose a thread **(formerly called a conversation)** in Codex.

历史上叫 conversation，后来改名 thread；CLI 面向用户仍说 "session"（`codex resume` 的帮助文本是 "Resume a previous interactive session"）。

### Session 结构体是什么

`Session` 是 thread 的**内存运行时**，`CodexThread` 只是它的薄封装：

```rust
pub struct CodexThread {
    pub(crate) session: Arc<Session>,     // ← 真正的运行时
    pub(crate) io: SessionIo,
    pub(crate) session_source: SessionSource,
    startup_metadata: ThreadStartupMetadata,
    rollout_path: Option<PathBuf>,        // ← 指向持久文件
    ...
}
```

`Session` 文档注释定义了它的边界：

> A session has at most 1 running task at a time, and can be interrupted by user input.

即**一个 thread 同时最多跑一个 turn**。它持有 `thread_id`、`active_turn`、`input_queue`、`services: SessionServices`（extensions、MCP runtime、provider 等）。

### turn 的边界

`core/src/client.rs` 头注释：

> A `ModelClientSession` is created per turn and is used to stream **one or more** Responses API requests during that turn.

所以 turn ≠ 一次模型请求：

```text
用户输入 → 模型请求① → 工具调用 → 工具结果回灌 → 模型请求② → ... → 无更多工具调用 → turn 结束
```

turn 级状态挂在 `TurnContext` + `turn_store`；turn 结束时 `TurnLifecycleContributor::on_turn_stop` 被调用，`turn_store` 被丢弃。

---

## 12. 退出、clear、resume、fork

### 退出 CLI：thread 结束，但数据不丢

退出走 `core/src/session/mod.rs` 的 `shutdown_and_wait()`：

```rust
pub(crate) async fn shutdown_and_wait(&self) -> CodexResult<()> {
    let session_loop_termination = self.session_loop_termination.clone();
    match self.submit(Op::Shutdown).await { ... }
    session_loop_termination.await;   // 等 session loop 真正退出
    Ok(())
}
```

在此之前 `on_thread_stop` 被调用，契约要求：

> Contributors must cancel and join their background work before returning.

但 **rollout 是增量落盘的**，不是退出时才写。进程退出只是"运行时结束"，对应 rollout 文件完好，下次可 resume。

### `/clear`：不是结束会话，是开一个新 thread

`tui/src/chatwidget/slash_dispatch.rs`：

```rust
SlashCommand::Clear => {
    self.app_event_tx.send(AppEvent::ClearUi { name: None });
}
```

`tui/src/app/event_dispatch.rs`：

```rust
AppEvent::ClearUi { name } => {
    self.clear_terminal_ui(tui, /*redraw_header*/ false)?;
    self.reset_app_ui_state_after_clear();
    self.start_fresh_session(
        tui, app_server,
        Some(ThreadStartSource::Clear),   // ← 关键
        /*initial_user_message*/ None, name,
    ).await;
}
```

`ThreadStartSource` 只有两个取值，映射关系：

```rust
ThreadStartSource::Startup => InitialHistory::New,
ThreadStartSource::Clear   => InitialHistory::Cleared,
```

所以 `/clear` = **清屏 + 起一个全新的 thread（历史为空）**。旧 thread 不会被删除，rollout 还在磁盘上，随时能 resume。

### resume：读出 rollout，重建运行时

```rust
pub enum InitialHistory {
    New,                      // 全新，无历史
    Cleared,                  // /clear 后的新 thread
    Resumed(ResumedHistory),  // resume 已有 thread
    Forked(Vec<RolloutItem>), // fork 出副本
}

pub struct ResumedHistory {
    pub conversation_id: ThreadId,
    pub history: Arc<Vec<RolloutItem>>,
    pub rollout_path: Option<PathBuf>,
}
```

CLI 侧（`cli/src/main.rs`）三种用法：

```rust
// Resume a previous interactive session
// (picker by default; use --last to continue the most recent)
Resume(ResumeCommand),

interactive.resume_picker = resume_session_id.is_none() && !last;
interactive.resume_last = last;
interactive.resume_session_id = resume_session_id;
```

- `codex resume` → 打开 picker 选一个
- `codex resume --last` → 接最近一个
- `codex resume <session-id>` → 指定

拿到 `InitialHistory::Resumed` 后 ThreadManager 用它重建 thread，并触发 `ThreadLifecycleContributor::on_thread_resume`——这就是 memories 扩展能在 resume 后重新水合 `thread_store` 的原因。

因为 `ThreadId` 持久、`Session` 是运行时构造的，所以 **resume 本质上是：同一个 thread 换了一个新的 `Session` 实例**。

### 四种启动方式对照

| 来源 | `InitialHistory` | turn 计数 | 是否复用原 rollout |
|---|---|---|---|
| 直接启动 `codex` | `New` | 从 0 开始 | 新建文件 |
| `/clear` | `Cleared` | 从 0 开始 | 新建文件（旧的保留） |
| `codex resume` | `Resumed` | 从上次继续 | ✅ 追加到原文件 |
| `codex fork` | `Forked` | 从 0 开始但带历史 | 新建文件，记录 `forked_from_id` |

`InitialHistory::forked_from_id()` 对 `New` 和 `Cleared` 返回 `None`——因为这两个在 code 看来"没有来源"，只有 `Forked` 记录父 thread id。

### 真正"删除"是哪条路径

`/delete`，对应 `AppEvent::DeleteCurrentThread`（`tui/src/chatwidget/slash_dispatch.rs`）。

---

## 13. 关键文件索引

### 扩展机制

| 文件 | 内容 |
|---|---|
| `codex-rs/ext/extension-api/src/contributors.rs` | 13 个 contributor trait 定义 |
| `codex-rs/ext/extension-api/src/registry.rs` | `ExtensionRegistry` / `ExtensionRegistryBuilder` |
| `codex-rs/ext/extension-api/notes.md` | 作者的能力→contributor 类型对照表 |
| `codex-rs/ext/extension-api/src/state.rs` | `ExtensionData`（三层 store） |
| `codex-rs/app-server/src/extensions.rs` | `thread_extensions()` — 唯一装配点 |

### 上下文

| 文件 | 内容 |
|---|---|
| `codex-rs/core/src/context/mod.rs` | 全部 context fragment 的导出清单 |
| `codex-rs/core/src/context/world_state/mod.rs` | `WorldState` / `WorldStateSection` / merge patch |
| `codex-rs/core/src/context/memory.rs` | `MemoryContextFragment`（8.9KB 上限） |
| `codex-rs/core/src/session/world_state.rs` | world state 实际构建与 contributor 调用点 |

### 记忆

| 文件 | 内容 |
|---|---|
| `codex-rs/memories/read/src/lib.rs` | 读路径入口、`memory_root()` |
| `codex-rs/memories/read/src/citations.rs` | `<oai-mem-citation>` 解析 |
| `codex-rs/memories/read/src/usage.rs` | 记忆使用埋点分类 |
| `codex-rs/memories/write/src/lib.rs` | 写路径常量（阶段、限额、目录） |
| `codex-rs/ext/memories/src/extension.rs` | 记忆扩展的 contributor 实现 |
| `codex-rs/ext/memories/src/local/search.rs` | 记忆检索（子串/逐行匹配） |
| `codex-rs/ext/memories/templates/memories/read_path_v2.md` | V2 记忆提示词 |

### 技能

| 文件 | 内容 |
|---|---|
| `codex-rs/skills/src/lib.rs` | 内置技能安装、指纹 marker |
| `codex-rs/ext/skills/src/lib.rs` | 技能扩展入口 |
| `codex-rs/ext/skills/src/dynamic_skill_selector.rs` | `CheapSkillSelector` trait 与候选打分器 |
| `codex-rs/ext/skills/src/catalog.rs` | 技能目录 |

### 插件与 MCP

| 文件 | 内容 |
|---|---|
| `codex-rs/plugin/src/manifest.rs` | plugin manifest 结构 |
| `codex-rs/core-plugins/src/lib.rs` | marketplace、安装、remote plugin |
| `codex-rs/codex-mcp/src/lib.rs` | MCP 主入口与导出 |
| `codex-rs/codex-mcp/src/connection_manager.rs` | 连接生命周期（改 MCP 优先动这里） |
| `codex-rs/ext/mcp/src/lib.rs` | MCP 扩展接入 |

### 会话与持久化

| 文件 | 内容 |
|---|---|
| `codex-rs/core/src/session/session.rs` | `Session` 结构体 |
| `codex-rs/core/src/codex_thread.rs` | `CodexThread` |
| `codex-rs/core/src/thread_manager.rs` | `ThreadManager` / `StartThreadOptions` |
| `codex-rs/history/src/lib.rs` | `InitialHistory` / `ResumedHistory` |
| `codex-rs/thread-store/src/lib.rs` | thread 持久化接口 |
| `codex-rs/core/src/compact.rs` | 压缩 |
| `codex-rs/core/src/client.rs` | 模型客户端（turn 生命周期） |
| `codex-rs/features/src/lib.rs` | `Feature` 枚举（全部功能开关） |
