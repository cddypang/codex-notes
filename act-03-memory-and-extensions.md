# 第三幕：记忆与扩展 —— 让能力可增长

> **幕目标**：讲清"持久化 → 扩展框架 → 四类具体能力"这条链路。
> 这一幕是 Codex 架构最有特色的部分：用一个编译期插槽框架，
> 让 memory / skill / MCP / plugin 都不必长进 core。
>
> **前置依赖**：第一幕（上下文、world state）与第二幕（工具抽象、审批）。
>
> **状态**：大纲（待展开正文）

---

## 目录

- [9. 持久化与恢复](#9-持久化与恢复)
- [10. Contributor 框架](#10-contributor-框架)
- [11. Skill](#11-skill)
- [12. Memory](#12-memory)
- [13. MCP + Plugin + Apps](#13-mcp--plugin--apps)
- [幕末收获](#幕末收获)

---

## 9. 持久化与恢复

### ① 问题

进程退出后到底发生了什么？"恢复一条会话"恢复的是哪些东西？

### ② 代码入口

```text
codex-rs/rollout/src/{lib,list,compression}.rs
codex-rs/thread-store/src/lib.rs
codex-rs/history/src/lib.rs                     # InitialHistory / ResumedHistory
codex-rs/core/src/session/rollout_reconstruction.rs
codex-rs/state/src/lib.rs                       # SQLite 元数据镜像
codex-rs/core/src/session/mod.rs                # shutdown_and_wait
```

### ③ 关键设计

- `RolloutItem` 的类型清单（SessionMeta / 消息 / 工具调用 / world state snapshot……）
- **增量落盘**：不是退出时才写，所以崩溃也不丢
- `InitialHistory` 四种形态：`New` / `Cleared` / `Resumed` / `Forked`
- `ResumedHistory { conversation_id, history, rollout_path }`
- `ThreadLifecycleContributor::on_thread_resume`：扩展如何重新水合 `thread_store`
- resume 的本质：同一个 `ThreadId` + 新的 `Session` 实例
- `/clear` 的真实行为（`ThreadStartSource::Clear` → `InitialHistory::Cleared` → 新 thread）
- fork 与 `forked_from_id`
- `state` crate 为什么存在（rollout 是真相源，SQLite 只是查询镜像）

### ④ 取舍

- 为什么用 JSONL 而不是直接上数据库
- 压缩 rollout（`LocalThreadStoreCompression`）带来的兼容性约束

---

## 10. Contributor 框架

### ① 问题

能力越加越多，如果都写进 `codex-core`，这个 crate 会变成什么样？
（AGENTS.md 里那句 "resist adding code to codex-core" 就是这条设计的动机。）

### ② 代码入口

```text
codex-rs/ext/extension-api/src/contributors.rs   # 13 个 trait
codex-rs/ext/extension-api/src/registry.rs       # ExtensionRegistry / Builder
codex-rs/ext/extension-api/src/state.rs          # ExtensionData（三层 store）
codex-rs/ext/extension-api/notes.md              # 作者的能力→类型对照表
codex-rs/app-server/src/extensions.rs            # 唯一装配点
```

### ③ 关键设计

- 依赖倒置：core 只认识 trait，不认识 memory / skill / MCP
- 13 个 trait 的职责划分（按生命周期阶段，逐个讲"什么时候被调用"）
- registry 的两种消费语义：**广播**（如 context）vs **认领**（如 `decide_approval`）
- 三层 store：`session_store` / `thread_store` / `turn_store`，以及"不能持有会话状态"的契约
- `TurnStartAdmission`（turn 准入门）与 `ExtensionEventSink`（事件出口）
- 装配点 `thread_extensions()`：一个函数决定一个会话装了哪些能力
- core 侧的消费示例：`session/world_state.rs` 遍历 `context_contributors()`
- 一次 turn 里 contributor 的完整调用顺序图

### ④ 取舍

- 编译期 trait（深但封闭）vs 运行期插件（浅但开放）
- **为什么不给扩展改主循环的能力**（对比 LangGraph 的开放图）
- contributor ≠ plugin，contributor ≠ hook（两个高频混淆点）

---

## 11. Skill

### ① 问题

怎么给模型"加一份专业知识"或"一套工作流程"，而不改一行代码？

### ② 代码入口

```text
codex-rs/skills/src/{lib,parser,loading,interface,model,mentions}.rs
codex-rs/ext/skills/src/{catalog,selection,invocation,render,world_state}.rs
codex-rs/ext/skills/src/dynamic_skill_selector.rs
codex-rs/skills/src/assets/samples/            # 内置技能
```

### ③ 关键设计

- `SKILL.md` 格式：frontmatter + 正文 + `references/` / `scripts/` / `assets/`
- 内置技能**编译进二进制**（`include_dir!`）+ 指纹 marker 避免重复写入
- 两类来源：Host skill 与 Executor skill（`sources.rs` / `provider/`）
- `SkillCatalog` / `SkillAuthority` / `SkillPackageId`
- 显式调用（`skill://` 路径、`$name` 提及、`UserInput::Skill`）与隐式调用检测
- 上下文注入与预算（`include_instructions` / `skill_max_context_tokens`）
- 动态选择器：`CheapSkillSelector` trait + BM25 / character n-gram / RRF
- **为什么它是影子实验**（只记录指标，不改模型可见 catalog）

### ④ 取舍

- 为什么技能是"提示词 + 资源"而不是可执行插件
- **为什么 Codex 没有向量 RAG**——在这里给出全仓库证据，并列出五个词法替代方案

---

## 12. Memory

### ① 问题

模型怎么记住跨会话的用户偏好、项目约定、上次的决定？

### ② 代码入口

```text
codex-rs/memories/read/src/{lib,citations,usage}.rs
codex-rs/memories/write/src/{lib,phase1,phase2,storage,workspace,guard}.rs
codex-rs/ext/memories/src/{extension,local,tools}/
codex-rs/memories/write/templates/memories/
```

### ③ 关键设计

读路径：

- `ContextContributor` 注入 memory summary（V2 按 8_900 字节切片）
- `ToolContributor` 注册 `memories` namespace：`add_ad_hoc_note` / `list` / `read` / `search`
- 引用机制：`<oai-mem-citation>` 块，`memory_summary.md` 不可引用
- 埋点：`usage.rs` 解析 shell 命令，判断模型实际读了哪个文件

写路径：

- Phase 1 抽取：`reasoning_effort = Low`、并发 8、最多扫 5000 thread、占上下文 70%
- Phase 2 合并：`reasoning_effort = Medium`、先算 workspace diff（≤4MB）
- 目录布局：`raw_memories.md` / `rollout_summaries/` / `extensions/`
- ad-hoc notes：用户在对话中提出"记住/忘掉"时的唯一合法写入点
- 安全保障：`guard.rs`、拒绝符号链接 root、extension 资源 7 天保留
- feature 开关：`MemoryTool` / `ExternalAgentMemoryImport` / `Chronicle`

### ④ 取舍

- 为什么用 LLM 抽取而不是向量检索（与第 11 节呼应）
- 幻觉与时效风险：提示词里"memory 不是当前行为的证明"这条规则
- 后台管线的成本（token、耗时）与收益

---

## 13. MCP + Plugin + Apps

### ① 问题

怎么接外部系统里的数据和动作（GitHub、Slack、Google Drive、自建服务）？

### ② 代码入口

```text
codex-rs/codex-mcp/src/{lib,catalog,connection_manager,binding,tools,resource_client}.rs
codex-rs/codex-mcp/src/mcp/auth.rs
codex-rs/rmcp-client/
codex-rs/ext/mcp/src/{lib,plugin,plugin_contributor,stream_manager}.rs
codex-rs/plugin/src/manifest.rs
codex-rs/core-plugins/src/{lib,manager,marketplace}.rs
codex-rs/connectors/src/lib.rs
```

### ③ 关键设计

MCP：

- 连接生命周期（`connection_manager.rs`——AGENTS.md 指定改 MCP 工具调用优先动这里）
- 工具发现、`McpBinding` / `PreparedMcpCall`
- 延迟加载：`defer_loading` + `tool_search`，避免工具表撑爆上下文
- 认证：OAuth / `auth_elicitation.rs` / `trusted_access.rs` / `UseXaa`
- resources：列表 / 读取 / 分页 / 事件流
- 协议版本开关：`Mcp20260728`、`CodexAppsMcp20260728`

Plugin：

- manifest 结构：`paths.{skills,onboarding_skill,mcp_servers,apps,hooks}` + `interface`
- **声明式资源包，没有代码注入点**
- 可执行能力只有两条路：hooks（有 allowlist）与 MCP servers
- marketplace / 安装 / 升级 / remote plugin / 推荐插件
- `request_plugin_install` 审批流

Apps / Connectors：

- 从 ChatGPT 后端拉应用目录（1 小时磁盘 + 内存缓存）
- apps 如何变成工具（Codex Apps MCP server）
- `AppToolPolicy`：哪些 app 对当前账号可用

### ④ 取舍

- 为什么不给插件代码执行能力（安全取向）
- 外部工具带来的 prompt injection 面，以及 Codex 用什么手段收敛
- 为什么"私有数据走连接器，不走 web search"

---

## 幕末收获

读完这一幕，你应该能：

- 说清 resume 到底恢复了什么
- 解释 contributor 框架解决了什么问题、放弃了什么
- 判断一个新能力该用哪种扩展形态（contributor / skill / plugin / MCP）

## 下一幕引子

能力齐了，但会话会越来越长，上下文窗口迟早会满；单个 agent 也不够用。
下一幕解决"长会话"和"规模化"。

→ [第四幕：长会话与规模化](act-04-long-session-and-scale.md)
