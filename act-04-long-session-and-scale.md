# 第四幕：长会话与规模化 —— 成熟智能体

> **幕目标**：讲清长会话下的上下文管理、多智能体拓扑，
> 以及主循环之外的两种执行形态。
>
> **前置依赖**：全部前三幕。特别是第一幕第 5 节（world state）——压缩与它强耦合。
>
> **状态**：第 14–16 节与全系列收束正文完成

---

## 目录

- [14. Compaction](#14-compaction)
- [15. 多智能体](#15-多智能体)
- [16. Code Mode 与 Realtime](#16-code-mode-与-realtime)
- [幕末收获](#幕末收获)

---

## 14. Compaction

### ① 问题

第三幕结束时，一个 turn 已经可以调用本地工具、MCP、Skill 和 Memory。问题是：
这些能力越好用，用户越倾向于在同一个 thread 里做很久。

假设一个修 bug 的会话已经经历：

- 40 轮对话；
- 几百次工具调用；
- 多次失败尝试和日志；
- 一次模型切换；
- 一个尚未完成的 turn。

窗口快满时，直接删除最早消息会让模型忘记目标和约束；把所有内容机械截断，又可能留下
半条 tool call。Compaction 要做的不是“少发一点”，而是把旧历史转换成一份新的、
仍然可继续推理的上下文起点。

本节的“压缩”专指**语义压缩**：生成 summary 或建立新 context window。第三幕 §9 的
`.jsonl.zst` 只是存储编码压缩，解压后历史完全不变，两者不能混淆。

### ② 代码入口

先看触发，再看实现选择。

#### 触发 A：普通 turn 开始前

`RegularTask` 进入 `run_turn` 后，在正式采样前执行：

```text
core/src/session/turn.rs:1298
  run_pre_sampling_compact
    ├─ maybe_run_previous_model_inline_compact
    │    ├─ comp hash 变化？
    │    └─ 切换到更小 context window 且新窗口装不下？
    └─ context_window_token_status().token_limit_reached
         └─ run_auto_compact(
              InitialContextInjection::DoNotInject,
              CompactionPhase::PreTurn,
            )
```

pre-turn 的压缩发生在普通 sampling step 创建之前。若上一轮用的是另一个模型，
Codex 会优先用**上一轮模型**压缩上一轮历史；只有压缩失败且 provider 支持时，才回退到
当前模型。这避免拿新模型去重写一个由旧模型产生、尚未兼容的上下文。

#### 触发 B：turn 中途、准备下一圈时

接点是第二幕 §7 的工具结果回灌：

```text
core/src/session/turn.rs:601
  needs_follow_up = 模型还要调用工具 || 有 pending input
  should_roll_over = needs_follow_up
      && (新的 context window 被请求 || token_limit_reached)

should_roll_over
  → run_auto_compact(
      InitialContextInjection::BeforeLastUserMessage { .. },
      CompactionPhase::MidTurn,
    )
  → continue 外层 turn loop
```

若模型已经给出最终回答、没有 follow-up，则不会走 mid-turn 分支。这里不能在
“回答已经完成”后强行插一个压缩 turn；只有准备继续工作时，才值得为下一圈重建上下文。

另一个 mid-turn 入口是 Guardian 证据导致 complete request 超限：普通估算尚未触发，
但实际请求仍然 `ContextWindowExceeded`。代码允许每个 model step 尝试一次 summarizing
compaction；token-budget reset 不走这个例外，因为重置会丢掉它试图保留的证据。

#### 触发 C：turn 结束后的提前压缩

如果最终回答已经完成，但配置了 `model_post_turn_compact_threshold_percent`，且 token
使用达到该比例，Codex 可在 post-turn 执行一次压缩：

```text
core/src/session/turn.rs:718
  最终回答已完成
    && 未启用 TokenBudget
    && token_status.turn_end_compaction_threshold_reached
    && 没有 pending input
    && 未取消
    → run_auto_compact(DoNotInject, CompactionPhase::PostTurn)
```

post-turn 失败只告警，不抹掉已经完成的回答。它的目标是让**下一次**用户输入到来时
已经有一个更小的历史，而不是延迟当前回答。

#### 触发 D：用户手动 `/compact`

手动压缩走 `CompactTask`：

```text
core/src/tasks/compact.rs:20
  CompactTask::run
    ├─ TokenBudget feature → run_manual_compact_task
    ├─ provider RemoteCompactionSupport::V2 → run_remote_compact_task
    └─ Unsupported → run_compact_task(summary prompt)
```

它有自己的 turn started / completed 生命周期，也复用同一套 pre/post compact hooks。

#### 统一分派：`run_auto_compact`

自动路径最后都进入 `core/src/session/turn.rs:1471`：

```text
run_auto_compact
  ├─ Feature::TokenBudget
  │    → compact_token_budget::run_inline_auto_compact_task
  ├─ RemoteCompactionSupport::V2
  │    → compact_remote_v2::run_inline_remote_auto_compact_task
  └─ RemoteCompactionSupport::Unsupported
       → compact::run_inline_auto_compact_task
```

### ③ 关键设计

#### a. “是否超限”不是单一 token 数

`context_window_token_status`（`core/src/session/context_window.rs:53`）同时计算：

- `active_context_tokens`：完整活跃上下文；
- `auto_compact_scope_tokens`：按配置 scope 计入自动压缩的 token；
- `auto_compact_scope_limit`：模型或配置的软阈值；
- `full_context_window_limit`：模型硬窗口；
- `buffered_auto_compact_limit`：软阈值加 fallback buffer；
- `turn_end_compaction_threshold_reached`：post-turn 提前压缩条件。

scope 可以是 `Total`，也可以是 `BodyAfterPrefix`。后者用当前 window 的首个 server
usage snapshot 作为 prefill baseline，只计算 prefix 之后增长的部分。这样，巨大的稳定
system/developer 前缀不会每轮都被当成“历史增长”。

超限判定是：

```text
scoped tokens >= soft limit + buffer
  或
active tokens >= model full context window
```

full window 始终是硬上限，不能被自定义 auto-compact scope 放大。

#### b. 三种实现：summary、remote v2、budget reset

| 实现 | 产物 | 何时使用 | 主要特点 |
|---|---|---|---|
| local Responses summary | 本地生成的 summary + recent user messages | provider 不支持 remote compaction | 兼容性最广，可由用户 prompt 控制 |
| remote compaction v2 | server 返回的 `Compaction` item | provider 明确声明 V2 | server 参与压缩，保留形状与模型训练更一致 |
| token-budget reset | 新的 context window | `Feature::TokenBudget` | 不调用 summary 模型，直接重开窗口 |

第三种仍被建模为 compaction，因为 UI、hooks、`ContextCompaction` turn item 和持久化
都需要看到同一种生命周期。实现上它调用 `Session::start_new_context_window`
（`core/src/session/mod.rs:4527`），而不是伪装成“没有发生压缩”。

#### c. 本地 summary 路径：先让模型总结，再构造 replacement history

`run_compact_task_inner_impl`（`core/src/compact.rs:245`）的核心顺序是：

```text
1. 发 ContextCompaction item started
2. clone 当前 history
3. 把 summarization prompt 作为 user input 追加
4. 发一次普通 Responses 请求，取最后的 assistant summary
5. 收集原历史中的真实 user messages
6. build_compacted_history(recent users + summary)
7. advance context window number / ids
8. 视 phase 决定是否重新注入 initial context
9. replace_compacted_history
10. recompute token usage
11. 发 completed item + “长 thread 可能降低准确性”的 warning
```

recent user messages 的预算默认是
`COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000`。从尾部向前选，超出预算的最后一条会被截断；
图片和音频在回退成纯文本时不会继续复制，避免旧的媒体附件重新占满窗口。

summary 不是一条 assistant message，而是 `CompactionSummary` fragment
（`core/src/context/compaction_summary.rs:5`），role 为 `user`，正文以
`SUMMARY_PREFIX` 开头。这个编码不是随意选择：后续恢复和下一次压缩要靠前缀识别
“这是 summary，不是真实用户请求”。

如果压缩请求本身也遇到 context window exceeded，而输入还多于一条历史 item，
本地实现会从最老 item 开始移除并重试。它保留的是“能总结多少就总结多少”，而不是因为
历史本身太大就完全无法压缩。

#### d. remote v2 路径：server 返回一个 compaction item

remote v2 的入口是 `core/src/compact_remote_v2.rs:81` / `:107`。它要求响应流中：

- 必须以 `response.completed` 结束；
- 必须恰好有一个 `ResponseItem::Compaction` 输出；
- 可以有其它辅助 output item，但不能出现第二个 compaction。

客户端收到 compaction output 后，不会只存这一项。`build_v2_compacted_history`
（`:504`）会从发送给 compaction 的输入里选择可保留内容：

- user / hook prompt；
- 非完成态的 agent message；
- 可选 client-authored developer message；
- 满足图片预算时保留图片。

保留总量上限是 64,000 tokens，单条 agent message 最多 10,000 tokens，最后再追加
server 返回的 compaction item。这样下一轮既拿到 server summary，也保留最近的关键输入和
多智能体协作证据。

remote v2 支持 previous-model fallback：若用上一轮模型压缩失败，`compact_model_fallback`
可记录原因并换当前模型重试一次。只有成功那次才会替换历史；失败不会留下半份 checkpoint。

#### e. pre-turn 与 mid-turn 为什么注入策略不同

`InitialContextInjection`（`core/src/compact.rs:66`）只有两种：

```text
DoNotInject
  → replacement history 只有 summary / retained items
  → reference_context_item = None
  → 下一次普通 turn 全量安装 initial context

BeforeLastUserMessage { world_state, step_context }
  → 立即构造 initial context
  → 插到最后一条真实 user message 之前
  → 保证 compaction summary 仍是 replacement history 的最后一项
```

pre-turn 和手动压缩还没有进入普通 sampling step。它们清理历史后，让下一次正常 turn
重新建立 context，行为更容易预测。

mid-turn 不同：模型已经在一个 turn 中途，刚刚收到工具结果且准备继续。主模型被训练成
在 mid-turn compaction 后把 summary 视为历史最后一项。若此时把 initial context 直接
追加到 summary 后面，顺序就与训练形态不一致。因此代码先构造 replacement history，
再把 initial context 插到最后一个真实 user / agent message 前；若只剩 summary，
则插到 summary 前。

`insert_initial_context_before_last_real_user_or_summary`
（`core/src/compact.rs:591`）的优先级依次是：

1. 最后一条真实 user / agent message 之前；
2. 没有真实消息时，summary 之前；
3. 没有 user 或 summary 时，最后一个 compaction item 之前；
4. 都没有时追加。

目标始终是：**compaction item 留在末尾，canonical initial context 在它之前重建。**

#### f. 替换历史时，world state 并不只是“被丢掉”

最终替换由 `Session::replace_compacted_history`
（`core/src/session/mod.rs:4027`）完成。它做三件事：

1. 给 replacement history 补齐 item id，并安装到 live `ContextManager`；
2. 构造持久化 `CompactedItem`，保存 summary、replacement history、window ids、
   retained context、Guardian checkpoint、token usage 和 resume metadata；
3. 写入新 checkpoint 对应的 world-state baseline、`TurnContext` 和 settings。

普通渲染出来的 world-state 文本会随旧 history 一起消失，但扩展可以实现：

```rust
fn retain_world_state_after_compaction(
    &self,
    previous_world_state: &serde_json::Map<String, Value>,
) -> serde_json::Map<String, Value>
```

（`ext/extension-api/src/contributors.rs:151`）。

扩展只返回自己 section 的稳定决策数据，不返回渲染文本。例如：

- skill 保留 cloud catalog 的预算 allocation；
- MCP 保留 resource origin / app attribution 等检查点；
- 权限系统用 `CompactPermissionsState` 保留新批准的 command prefixes，避免 summary
  丢掉“这条命令前缀已经被批准”的事实。

下一次 step 的 `build_world_state_for_step` 会把这些元数据当作 previous state，
重新采集完整状态并只渲染必要差异。因此 compact 不是简单 `history.clear()`，而是
“重建历史 + 保留不可从文本恢复的决策状态”。

#### g. context window 是可追踪对象，不只是一个整数

每次成功压缩都会推进 `AutoCompactWindowIds`：

```text
window_number += 1
previous_window_id = old window_id
window_id = new UUIDv7
first_window_id 保持不变
```

这些 id 会写进 `CompactedItem` 和 `TurnContext`，也会通过 `TokenBudgetContext` 出现在
模型上下文与 world state 中。它们支持：

- 区分同一个 thread 的不同逻辑窗口；
- 多智能体和 rollout budget 归因；
- resume 时恢复 window 边界；
- 对旧 compaction 检查点做兼容回放。

所以“压缩后还在同一个 thread”与“仍然在同一个 context window”是两件事。

#### h. token budget 与 rollout budget 控制不同资源

`session/token_budget` 管理当前 context window：

- 在剩余 token 低于模型阈值时注入 reminder；
- 到零时可注入一次 auto-compact fallback prompt；
- `Feature::TokenBudget` 下压缩直接开新 window，不生成摘要；
- 相关 prompt 不是每轮重复注入，而是在 window 内 claim 一次。

`rollout_budget` 则是整个 agent tree / rollout 的共享加权预算。子 agent 的模型调用、
工具结果和 compaction 会把 usage 汇总到 `AgentControl`；root 在合适的 step 注入
`<rollout_budget>` reminder。它回答“这棵树总共还能花多少”，context token budget
回答“当前这个模型请求还剩多少”。

#### i. hooks 围绕压缩生命周期，而不是围绕“摘要函数”

三种实现都执行：

```text
PreCompact hook
  → Continue：开始压缩
  → Stopped：TurnAborted

执行 compression

成功后 PostCompact hook
  → Continue：正常返回
  → Stopped：TurnAborted
```

这使 hook 看到的是“发生了自动 / 手动压缩”这一事实，不必知道本次是 local summary、
remote v2 还是 token-budget reset。pre-turn 失败会保留 incoming prompt；post-turn 失败
会保留已经完成的回答。压缩失败不应顺便吞掉用户输入。

### ④ 取舍

**压缩是不可逆的信息损失，哪些内容必须留下？**

当前策略保留四类东西：

1. 最近 user messages：当前意图和最直接的约束；
2. summary：旧任务的压缩记忆；
3. retained context / Guardian checkpoint / tool provenance：模型文本无法可靠恢复的决策；
4. world-state baseline：权限、工具、环境等下一轮仍需要比较的状态。

代价是 replacement history 仍可能很长；20k user-message 预算、64k remote retention
和 summary 本身都要占窗口。Codex 接受这种冗余，因为“保住当前任务可执行性”比追求
最少 token 更重要。

**为什么 pre-turn 和 mid-turn 不共用同一种历史形状？**

pre-turn 是新的请求边界，可以在下一次 sampling 前完整重装 context；mid-turn 已经在
模型训练所假设的“工具调用后继续”路径里，summary 必须保持最后一项。如果统一成一种形状，
要么每个 mid-turn 都多发一份 initial context，破坏训练分布；要么 pre-turn 少发 context，
让权限和工具状态缺失。

**local summary 与 remote v2 的区别是什么？**

local summary 是 Codex 自己发一次普通 Responses 请求，控制点清楚、兼容 provider 多，
但摘要格式和模型输入由客户端构造。remote v2 让 provider 直接返回 compaction item，
更接近服务端训练过的压缩接口，并能利用 server 侧保留策略；代价是 provider 能力、
协议版本和输出数量校验更严格。

两条路都复用了同一个 replacement-history / window / hook / persistence 壳，而不是各自
实现一套 turn 生命周期。

**token-budget reset 为什么也算 compaction？**

它不做摘要，信息损失更大，但目的仍是“让长 turn 继续”。如果给它另造一种事件，UI、hooks、
预算和 resume 都要区分第四种路径；复用 compaction 生命周期可以保持外部行为一致。

代价是模型必须明白窗口已经换了，且旧历史不能假设仍在当前上下文里。因此系统需要
`TokenBudgetContext`、window ids、reminder 和 fallback prompt 一并参与，而不是只清空 history。

回到主流程：成功压缩后，`run_turn` 继续使用同一个 `TurnContext` 和新 history；下一次
sampling 会看到 summary、保留的用户消息和重建后的 initial context。单条 thread 因此可以
跨过窗口边界。但当一个任务线本身需要并行推进时，还需要下一节的多智能体树。

---

## 15. 多智能体

### ① 问题

§14 让一条 thread 能跨过窗口继续工作，但没有改变一个事实：所有推理都发生在同一条
上下文里，所有工具调用也围绕同一个 agent 目标推进。

如果用户说：

> 同时查清认证、数据库和前端三个模块，再汇总修改方案。

一个 root agent 可以做，但它要依次探索三个区域，把三份调查路径全部留在自己的 history。
更合理的方式是：

- root 保留目标和汇总权；
- 三个子 agent 各自只带必要的上下文去调查；
- 子 agent 可以并行执行；
- 最终只把有界结果送回 root。

这时要解决的不只是“多开一个模型请求”，而是：

- 新 agent 的身份、路径和父子关系；
- 子 agent 能继承什么配置、不能继承什么权限；
- 一次 spawn 是 fresh、full-history fork 还是最近 N turn fork；
- 消息、follow-up、interrupt、completion 各是什么语义；
- 子 agent 太多时怎样限流、卸载和恢复；
- root 结束后怎样清理整棵树。

### ② 代码入口

所有协作工具仍从 §7 的 tool router 进入。`add_collaboration_tools`
（`core/src/tools/spec_plan.rs`）根据 `MultiAgentVersion` 注册不同工具面。

#### 场景 A：root 调 `spawn_agent`

以 MultiAgentV2 为例：

```text
core/src/tools/spec_plan.rs
  add_collaboration_tools
    └─ SpawnAgentHandlerV2

模型输出 spawn_agent
  → core/src/tools/handlers/multi_agents_v2/spawn.rs
       ├─ 解析 message / task_name / agent_type / fork_turns / model
       ├─ prepare_agent_spawn_config
       ├─ thread_spawn_source(... task_name)
       ├─ AgentControl::spawn(SpawnRequest)
       └─ emit SubAgentActivity::Started
```

`AgentControl` 是 controller 抽象（`core/src/agent/api.rs`）。本地实现负责：

```text
reserve agent slot/path
  → 准备 child config / session source
  → 启动 child thread
  → 追加到 parent 的 subagent registry
  → 提交第一条 child input
  → 启动 terminal-status watcher
```

child 的第一条输入不是普通 user message，而是带 author / recipient / message type 的
`InterAgentCommunication`。root 的 spawn tool call 本身也通过
`ToolCallRuntime` 执行，因此并发门、取消和工具结果回灌仍遵守第二幕。

#### 场景 B：root 与 child 通信

```text
send_message
  → target resolve
  → AgentMessage::Plaintext / Encrypted
  → InterAgentCommunication { trigger_turn: false }
  → child input queue

followup_task
  → target resolve
  → InterAgentCommunication { trigger_turn: true }
  → child 若 idle 则开始新 turn

wait_agent
  → 订阅当前 turn 的 InputQueueActivity
  → mailbox activity / steering / timeout
```

V2 的 `wait_agent` 不直接返回所有 child 的内容。它只告诉 root：

```text
message: "Wait completed."
timed_out: false
```

真正的 completion / follow-up 文本以 mailbox message 的形式进入 root 的历史。

#### 场景 C：child 完成

child turn 结束时会生成 `AgentTurnOutcome`。local `AgentControl::turn_finished`
（`core/src/agent/control/completion.rs`）：

```text
读取 child 的 ThreadSpawn source
  → 找到 parent thread / parent AgentPath
  → 生成 terminal result message
  → 以 trigger_turn=false 投递到 parent mailbox
  → 记录 agent-result trace
```

root 正在运行时可被 mailbox 唤醒；root idle 时消息保持为可控的 pending input，
是否开新 turn 由上层策略和 wait/turn 状态决定，而不是 child 直接启动 parent。

### ③ 关键设计

#### a. V1 与 V2 是两套协调语义

`MultiAgentVersion` 由 feature 和模型元数据共同决定：

```text
Feature::MultiAgentV2 → V2
Feature::Collab       → V1
都没有                → Disabled
```

两者共用一个底层 `AgentControl`，但对外工具不同：

| 能力 | V1 | V2 |
|---|---|---|
| 工具 namespace | `multi_agent_v1` | 默认 namespace 或配置的 `tool_namespace` |
| spawn 地址 | 返回 agent/thread id | 返回 canonical `task_name` |
| 延续 | `send_input` | `send_message`、`followup_task` |
| 等待 | `wait_agent` 直接看 agent 状态 | `wait_agent` 等 mailbox / steering 活动 |
| 中断 | interrupt/close 通过 V1 工具 | `interrupt_agent`，不自动关闭 agent |
| 列举 | 通常按 id 管理 | `list_agents` 按 task-path 前缀过滤 |
| 生命周期 | 显式 `resume_agent` / `close_agent` | residency 自动卸载，按 parent 恢复 |

V1 更像“管理一批 thread”；V2 更像“管理一棵有名字的任务树”。

#### b. `ThreadId` 是传输身份，`AgentPath` 是模型可读身份

child 仍有唯一 `ThreadId`，用于路由 `Op`、工具、rollout 和事件。但 V2 在它之上增加了
`AgentPath`：

```text
/root
/root/auth
/root/database
/root/frontend
/root/auth/patch
```

`task_name` 必须是稳定的 lowercase 路径片段。模型说 `send_message` 到
`database`，resolver 会相对当前 caller 的 path 解析；底层最终仍转换成 `ThreadId`。

这样做的收益是模型不必记 UUID，也能用路径前缀 `list_agents("/root/frontend")`。
代价是 path 必须全局保留、不能歧义；spawn 时要先 reserve，失败要释放，避免两个同名
child 争用一个路径。

`AgentMetadata` 同时保存：

```text
agent_id     = ThreadId
agent_path   = AgentPath
agent_role   = role name
agent_nickname
```

三者不是重复：id 用于执行，path 用于模型路由，role 用于配置与可观测性。

#### c. 父子拓扑由内存 registry 和持久 graph 共同表示

local agent runtime 在内存里维护 live agents、reserved path、status 和 parent-child 关系。
MultiAgentV2 另外把 spawned-thread edge 写入 `AgentGraphStore`：

```text
parent_thread_id
child_thread_id
status = Open | Closed
```

`AgentGraphStore` 是 storage-neutral trait，本地实现是 SQLite。它能：

- upsert 一个 child 的唯一父边；
- 更新 open/closed 状态；
- 列直接 children；
- 广度优先列 descendants，并可按 edge 状态过滤。

内存 registry 保存“现在加载了谁、谁在运行”；graph store 保存“即使 child 被卸载或
进程重启，这棵树原本是什么”。V2 resume 父 thread 时可以按 open edges 懒恢复 agent
metadata，而不会立刻重新启动所有 child runtime。

#### d. spawn 有三种上下文接法

V2 `spawn_agent` 的 `fork_turns` 支持：

- `none`：fresh child，只拿新的任务；
- `all`：full-history fork；
- 正整数 N：保留最近 N 个 user turn 的 fork。

full-history fork 会复制/引用父模型上下文，尽量保留 prompt prefix，使 child 的第一个请求
仍可能命中缓存。截断 fork 会丢掉部分 prefix，因此必须重建设置并清理不应继承的内容。

`keep_forked_rollout_item`（`core/src/agent/control/spawn.rs`）明确过滤：

- 保留 system / developer / user 消息和 completed final answer；
- 保留 compaction checkpoint、event boundary、必要 context baseline；
- 去掉旧 reasoning、普通 tool call/output、agent progress message；
- 不复制父 thread 的累计 token usage。

也就是说，child 继承的是“可继续工作的模型上下文”，不是父 thread 的完整运行时。

#### e. child config 是父权限的子集，不是父进程的副本

spawn 前 `prepare_agent_spawn_config` 以当前 step 的有效配置为基座，再叠加：

- role；
- 可选 model / reasoning effort；
- service tier；
- environment / cwd；
- exec policy；
- MCP client extensions。

权限应用有一条硬规则：

> role 可以收紧父会话能力，不能替换或扩大父会话的权威。

`apply_role_to_config` 会保留 approval policy、approvals reviewer、cwd 和 permission profile
的运行时约束；role 中只能关闭高风险 feature，例如 shell、apps、plugins、memory 或
request-permissions，不能借 role 文件重新打开父会话已禁用的能力。

executor 环境也需要与 parent 当前 ready environment 匹配；远程 executor 上权限变化不允许
通过简单“取交集”猜测，必要时拒绝 resume/spawn，而不是放宽。

#### f. agent role 是配置模板，不是自由提示注入

`codex-agent-roles` 从配置发现的 role 文件加载 typed role config。内置 role 包括：

- `default`：普通 agent；
- `explorer`：只做代码调查，鼓励并行；
- `worker`：执行实现、测试和重构。

role 可以设置 developer instructions、model、effort、summary、verbosity、personality、
service tier 和可关闭的 features/skills。`spawn_tool_spec` 把 role 描述与锁定设置告诉
模型，让 caller 知道哪些参数不可覆盖。

这不是直接拼接任意字符串：role 文件先解析为配置层，再由 `apply_role_to_config` 走与
普通 config 相同的验证/约束流程。

#### g. V1 的 resume/close 与 V2 的 residency 解决不同问题

V1 提供显式：

```text
spawn_agent → send_input / wait_agent → close_agent
                                  ↘ resume_agent
```

closed agent 不再占并发槽，但 rollout 仍可恢复；resume 读取原 thread history 并重新注册。

V2 暴露 `interrupt_agent` 而不是 `close_agent`。terminal 的 child 可以被 residency 策略
自动 eviction：

- 只清理 completed / errored / interrupted 且无 active turn / pending mailbox 的 agent；
- 先持久化 rollout、保存 environment selection，再 shutdown 并从 registry 移除；
- parent 之后的 send/follow-up 可通过 `ensure_v2_agent_loaded` 从 parent 权威下恢复 child。

这样模型不必手工管理“什么时候 close”，host 用 LRU 风格控制常驻线程数。代价是 agent
仍逻辑存在，只是 runtime 被卸载；“unloaded”不等于“completed”，恢复仍可能失败。

#### h. 并发同时受 depth、slot 和 execution limiter 约束

三层限制回答不同问题：

```text
agent_max_depth
  → 树最多能长多深

max_concurrent_threads_per_session / spawn slots
  → 一共允许多少 child 身份

AgentExecutionLimiter
  → V2 child 同时有多少个在真正执行 turn
```

V2 的 `max_concurrent_threads_per_session` 包含 root，所以有效 child 上限通常按
总数减一计算。residency 回收只作用于 idle terminal agent；正在 tool/approval 中的 child
不会被当作可卸载对象。

深度也不只防止成本：没有深度上限时，agent 可以递归 spawn 自己，形成无限任务树和消息回声。

#### i. 消息投递语义刻意分成“排队”和“触发”

`MessageDeliveryMode` 只有两个值：

```text
QueueOnly
  send_message / completion notification
  → 放入 mailbox，不主动开始 idle turn

TriggerTurn
  spawn initial task / followup_task
  → 若目标 idle 则启动新 turn；若正在采样则在边界投递
```

`DeliveryReceipt` 只表示目标 acceptor **接受了**消息，不表示模型已经读过或执行。
这种区分避免 caller 把“消息已入队”误判为任务完成。

child completion 使用 QueueOnly，避免 parent 已经结束时 child 再反向启动 parent；
`followup_task` 使用 TriggerTurn，但禁止直接 targeting root，防止 child 无限唤醒主任务。

V2 的 `wait_agent` 返回的也是“有 mailbox 活动”，而不是把内容复制一遍；真正的消息仍在
parent input queue/history 中，只消费一次。

#### j. 子 agent 的上下文是隔离的，但共享树级资源

隔离项：

- 各自的 thread history / context window；
- 各自的 turn、工具并发和 active task；
- 各自的 role instructions；
- 子 agent 的 reasoning 和中间工具输出不自动进入 parent history。

共享项：

- root `SessionId` 下的 agent tree 和 rollout budget；
- parent 派生的 exec policy / environment authority；
- host services、skills、plugin/MCP provider；
- 可选 message board、agent graph store；
- 同一进程内的 ThreadManager / model clients / storage handles。

所以多 agent 不是多个独立 Codex，而是同一 host 树里的多个受限 `Session`。

#### k. message board 是可选通信层，不是调度器

`Feature::AgentMessageBoard` + `Feature::MultiAgentV2` 打开一个 per-tree board：

- local SQLite；
- 训练/临时场景的 in-memory board；
- 配置了 endpoint 时的 remote board。

board 支持 posts、读取和 notification，但 `LocalBoardHost::notify` 只向已加载且在同一 tree
的 recipient 投递；recipient 不活跃时返回 `SkippedInactive`。它不会启动、恢复或排队等待
idle agent。

边界因此很清楚：

```text
AgentControl → 谁存在、谁运行、消息如何唤醒
MessageBoard → 树级共享信息面
```

message board 不能绕过 agent ownership 和 concurrency 规则。

#### l. completion notification 是结果回灌，不是共享整个 child transcript

child 的 completion message 只包含 terminal status（completed 文本、error 或 interrupted）
以及必要的 task path。child 的完整工具轨迹留在自己的 rollout；root 需要时可以：

- 通过 task path 发送 follow-up；
- 用 list/wait 找到 agent；
- 读取已存 child thread；
- 由 child 在消息里主动给出摘要。

这使 parent context 的成本有界，也让并行 worker 的中间噪声不会自动污染 root。

### ④ 取舍

**为什么多智能体默认不是无条件的？**

每个 child 都是真实 model request、真实工具执行面和真实持久化 thread。即使没有 token
成本，也会带来调度、权限、消息顺序和可观测性问题。因此 `Collab` / `MultiAgentV2`
必须是显式 feature；模型元数据、depth、槽位和 execution limiter 还会继续限制。

**为什么不让多个 agent 共享一个大 context？**

共享 context 等于共享历史写入权。两个 agent 并行追加消息时，prompt prefix 很快就会互相
破坏，工具结果也难以归因到某个 agent。当前设计让每个 child 有独立 history，只通过有类型
消息交换必要信息。

代价是需要显式任务拆分和汇总，root 不能“自动看到所有 child 的思考过程”。

**为什么 V2 用 task path 而不是只用 ThreadId？**

UUID 适合系统路由，不适合模型做多步协作。`/root/database` 这类路径能直接表达归属、
支持前缀列举，也让 follow-up 的目标在上下文中稳定可读。ThreadId 仍保留在底层，避免让
可读名称承担安全身份。

代价是 path 生命周期、唯一性和 resume 校验更复杂。

**V1 显式 close/resume 与 V2 residency 哪个更好？**

V1 的控制面直白，调用者明确知道 agent 何时关闭和恢复；V2 把常驻管理交给 host，
模型不需要把大量注意力放在 clean-up 上，但“逻辑存在 / 物理卸载”的状态更多。

当前没有用一套语义强行统一，而是按 feature/version 隔离，降低旧 rollout 和新任务树
互相污染的风险。

**与 supervisor / swarm 框架相比，这套结构是什么？**

Codex 不是让多个 agent 自由广播并自行收敛。它是一个 **host-owned root tree**：
root 通过工具显式 spawn、召集、等待和中断固定路径的 child；host 负责并发、权限、
持久化与 completion 路由。root 可以扮演 supervisor，但没有另一个独立的 supervisor
runtime 替它控制整个图。

这牺牲了 swarm 的开放拓扑，换来可审计的任务路径、有限的并发和一致的权限根。

回到主流程：§15 把“一个 thread”扩展成“一棵可控的 agent 树”。下一节再改变执行形态：
让模型写一段程序批量编排工具，或者通过 Realtime 通道边说边做。

---

## 16. Code Mode 与 Realtime

### ① 问题

§15 扩展了“谁来做”，但没有改变“怎么做”：模型仍然按 Responses 协议的
function/custom tool call，一次返回一批调用，拿到结果后再采样。

这适合开放式探索，却不适合所有任务。例如：

> 扫描 200 个文件，按正则筛选，把结果分批交给另一个工具，最后只告诉我哪三个文件需要改。

若每个筛选步骤都是一次 tool call，模型要反复看中间结果、重复解释控制流，消耗很多 round trip。
Code Mode 允许模型写一段 JavaScript，把循环、过滤、并发和后续工具调用放进一个执行单元。

另一个问题来自交互方式。用户可能希望直接说话、打断、听到即时回答，而不是等完整 turn
结束。Realtime 提供低延迟语音通道，但它最终仍要调用 Codex 的编码 agent 处理复杂任务。

所以本节不是再介绍两个工具，而是回答：

- 程序化工具编排何时进入主 turn；
- 嵌套工具调用怎样仍经过审批、并发和持久化；
- 语音通道怎样把任务 handoff 给原来的 agent，再把结果播回去。

### ② 代码入口

#### 场景 A：模型调用 Code Mode `exec`

```text
tool plan
  → core/src/tools/spec_plan.rs
       add_code_mode_executors
       register_code_mode_executors
  → CodeModeExecuteHandler
  → CodeModeService::execute
  → CodeModeSessionProvider
       ├─ ProcessOwnedCodeModeSession（独立 host 进程）
       └─ InProcessCodeModeSession（测试 / 内嵌运行）
```

`CodeModeExecuteHandler` 把当前 step 可见的 nested tools 转成
`CodeModeToolDefinition`，再把模型给出的原始 JavaScript source 和 yield 参数交给 runtime。

#### 场景 B：代码里再次调用工具

```text
V8 callback
  → CodeModeSessionDelegate::invoke_tool
  → CodeModeDispatchBroker
  → ToolCallRuntime
  → registry dispatch
  → 原有 handler
```

嵌套调用携带 `cell_id` 和 `runtime_tool_call_id`，在 core 内记为：

```text
ToolCallSource::CodeMode {
  cell_id,
  runtime_tool_call_id,
}
```

因此它仍是正常工具调用，不是 V8 直接访问文件系统或网络。

#### 场景 C：cell 没在 yield 时间内跑完

```text
exec returns RuntimeResponse::Yielded { cell_id }
  → 模型下一轮调用 wait({ cell_id, yield_time_ms, max_tokens })
       ├─ terminate=false → 继续观察
       └─ terminate=true  → 终止 cell
```

`wait` 是 Code Mode 自己的控制工具，不走普通 user-action pre/post hooks；但 cell 内发出的
真正工具调用仍逐次进入 normal dispatch。

#### 场景 D：启动 Realtime 会话

```text
app-server/tui
  → ConversationStartParams
       transport =
         Websocket
         | Webrtc { sdp }
         | ExistingCall { call_id, .. }
  → core/src/realtime_conversation.rs:handle_start
       ├─ 构建 backend prompt / startup context / session config
       ├─ RealtimeConversationManager::start
       └─ 发布 RealtimeConversationStarted
```

WebRTC 模式下，本地 `codex-voice-host` helper 负责麦克风、扬声器和 WebRTC 音频；
Codex 仍用 sideband WebSocket 收发控制事件、文本 transcript 和 handoff。

#### 场景 E：语音模型把任务交给编码 agent

```text
RealtimeEvent::HandoffRequested
  → RealtimeHandoffAdmission::route
       └─ Session::route_realtime_text_input
            └─ TurnInputMode::StartOrSteer

编码 agent 产生输出 / completion
  → RealtimeConversationManager::handoff_out / handoff_complete
  → RealtimeOutbound
  → Realtime 输入 task 发回语音会话
```

Realtime 不是另一套 Codex 主循环。它把请求送回同一个 `Session` 的现有 turn 入口。

### ③ 关键设计

#### a. Code Mode 是执行形态，不是万能工具

Code Mode 的价值来自**控制流留在模型生成的程序里**：

```javascript
const hits = await Promise.all(
  files.map(file => tools.exec_command({
    cmd: `rg -n "refund" ${file}`,
    workdir: repo
  }))
);

const changed = hits.filter(...);
text(JSON.stringify(changed));
```

模型不用先发一个 tool call、读取全部 JSON、再发下一个 tool call；它把“如何组合工具”
写成代码，由 runtime 执行。

但每个 `tools.exec_command` 仍然是一次独立 host tool call。Code Mode 减少的是模型采样轮次，
没有减少 host 的实际执行次数，也没有绕过审批或 sandbox。

#### b. 工具表被投影成 V8 API

`collect_code_mode_tool_definitions`（`codex-rs/tools/src/code_mode.rs`）把 function、
freeform、namespace tools 转成 Code Mode definitions：

```text
ToolName    → JS 可调用路径
schema      → 参数说明
description → 工具摘要
kind        → function / freeform 输入形式
```

名称必须稳定且唯一；namespace 用 `__` 或配置的间接前缀拼接。Code Mode 只投影当前 step
实际可用的 tools，因此 MCP catalog 或 plugin 变化仍由 §6、§13 决定。

`CodeModeOnly` 下，这些 nested tools 不再作为顶层 schema 暴露给模型，模型只看到
`exec` / `wait` 两个入口；工具 router 仍保留 handler，供 cell 内按名称调用。

#### c. V8 运行在受限 host 里，输入只有 source 与工具描述

`ProcessOwnedCodeModeSessionProvider` 默认寻找安装包里的 code-mode host executable。
host 通过 gRPC/remote session 提供逻辑 session；真正 JavaScript 由
`code-mode-runtime` 的 V8 执行。

`initialize_v8`（`code-mode-runtime/src/v8_init.rs:24`）初始化进程级 V8，并允许显式选择
`V8JitMode::Disabled`。禁用 JIT 时设置 `--jitless`；默认模式也关闭了当前 pinned V8
中与 mixed-element sort 相关的 Maglev/Turbo 路径。

cell 得到的环境不是完整 Node.js：

- 没有任意 module loader 访问系统文件；
- 只有 host 注入的 tool globals、`text` / image / audio 输出和 `notify`；
- 所有异步 callback 最终回到 Rust delegate；
- runtime 有 heap、yield 时间和输出等执行限制。

因此“模型能写 JS”不等于“模型获得新的 OS 权限”。它能调用的能力仍由当前 step 的
tool definitions 和第二幕的权限链决定。

#### d. `exec`、`wait` 与 cell 生命周期

一次 `exec` 创建 `CellId`。runtime 可能返回：

| 结果 | 含义 | 后续 |
|---|---|---|
| `Yielded` | 代码仍在运行，但有新输出或到了 yield 时间 | 模型调 `wait` |
| `Completed` | 脚本正常结束或返回 script error | 本次 cell 关闭 |
| `Terminated` | 被 `wait(terminate=true)` 或 turn interrupt 终止 | 本次 cell 关闭 |
| `Pending` | 某类观察边界已到，仍有 nested calls 未完成 | 主要在内部 dispatch 使用 |

`InProcessCodeModeSession` 的 `resolve_yield_timeout` 会给较长 yield 留 1 秒 grace time，
并受配置的最大 yield 时间限制。

cell 的输出分 text / image / audio，可复用第二幕的结果预算。脚本错误作为
`Script error:\n...` 回灌给模型，而不是直接终止 host；取消、runtime crash 或 cell 丢失
才属于更硬的失败。

#### e. nested tool 的审批、并发和持久化仍走主链

Code Mode 最容易误解的一点是“一次 exec 里面发生的调用是不是不可审计”。实际链路是：

```text
V8 tools.foo
  → delegate invoke_tool
  → broker 等待 cell dispatch gate
  → ToolCallRuntime
  → approval / execpolicy / sandbox
  → ToolOutput
  → JSON result 返回 JS
```

cell 内多个工具可继续利用第二幕的 read/write 并发门；非并行工具不会因为写在同一段 JS 里
就自动并行。每次 nested call 都有独立 call id、tool lifecycle 事件，并进入 Code Mode 的
executed-call 账本；它们不会伪装成模型直接发出的顶层 function call。模型看到的是外层
`exec/wait` 结果，而不是每个中间结果都单独进入下一轮 context。

broker 还解决 cell 跨 turn 边界的问题：cell callback 只持有所在 step 的弱引用；若 turn
已经结束，迟到的 nested work 会被丢弃或取消，不能在旧权限上下文里继续跑。

#### f. Code Mode 的 fallback 是显式策略

`requested_tool_mode` 先看模型元数据，再回退到 `Feature::CodeModeOnly` / `CodeMode`。
`effective_tool_mode` 再检查 host 是否可用：

```text
请求 CodeMode，但 code-mode host 不可用
  → 若允许 in-process fallback：Direct
  → 否则：保留 CodeMode，让后续 fail closed
```

若模型本身不声明 Code Mode 支持，`unsupported_code_mode_warning` 会提示性能可能下降。
`CodeModeOnly` 隐藏直接 tools，因此 host 不可用时不能静默改成普通工具模式；fail closed
比让模型调用不存在的工具更可靠。

#### g. Realtime 有三种 transport，但共享同一个会话模型

`ConversationStartTransport` 支持：

- `Websocket`：直接连 Realtime WebSocket；
- `Webrtc { sdp }`：本地 helper 生成 offer SDP，向 backend 换取 answer，音频走 WebRTC；
- `ExistingCall { call_id, .. }`：接入已存在的实时 call，不允许重设 prompt/model/voice 等。

WebRTC 的 native media 放在 `codex-voice-host` 子进程：

```text
RealtimeWebrtcSession::start
  → VoiceHost::connect
  → initialize_runtime
  → start_transport → offer SDP
  → apply_answer_sdp → devices open
```

握手成功只证明 helper 和 transport 兼容，不代表麦克风/扬声器已经可用。`apply_answer_sdp`
还会打开 devices 并恢复用户当前的 mute/suppress 状态。

#### h. Realtime 的 startup context 是“工作场景摘要”

`build_realtime_startup_context`（`core/src/realtime_context.rs:59`）在 Realtime 音频开始前
构造一份约 5,300 token 的背景：

| section | 预算 | 内容 |
|---|---:|---|
| Current Thread | 1,200 | 当前 thread 最近的 user/assistant turns |
| Recent Work | 2,200 | 最近 thread，按 repo / cwd 分组 |
| Machine / Workspace Map | 1,600 | cwd、git root、有限目录树 |
| Notes | 300 | 构建方式与限制 |

每轮 turn 还会截到约 300 token。目录树只递归两层、每层最多 20 项，并跳过
`node_modules`、`target`、`.git` 等噪声目录。

这份 context 明确排除 AGENTS、repo memory instructions 和 memory summary。Realtime 是
低延迟会话入口，不能把普通 turn 的全部大上下文直接塞进音频 session。

#### i. Realtime 的 history 是 sparse presentational history

`RealtimeHistoryState` 监听：

- conversation started / closed；
- input/output transcript delta / done；
- handoff；
- backing turn 的 item / agent message。

它把 transcript 封成有边界的 `RealtimeItem`，并写入 `RolloutItem::RealtimeItem`。
这些 item 用于 resume 后重建 UI transcript 和实时呈现，不是普通 model-visible history。

事件顺序由 `RealtimeEventOrder::BeforeEvent` / `AfterEvent` 控制；例如真实 user input 到达时，
先封存已经生成的 voice transcript，再记录 backing turn 的 user message，避免同一句话在
UI 和模型历史上重复。

#### j. handoff 把语音任务交回主 agent

Realtime 模型识别出需要编码 agent 时发出 `HandoffRequested`：

```text
input + optional transcript delta
  → RealtimeDelegation fragment
       <input>...</input>
       <transcript_delta>...</transcript_delta>

  → Session::route_realtime_text_input
       TurnInputMode::StartOrSteer
```

每个 delegation 字段最多 4 KiB，输入保留开头、transcript 保留尾部，并做 XML text escaping。
handoff 任务是普通 user turn input，`turn_trigger = "realtime"`，所以会遵守第一幕的 turn
准入门、第二幕的工具链和 §14 的上下文管理。

主 agent 的输出再通过 `handoff_out` 发回 Realtime：

- commentary / final answer 可分流到不同 channel；
- 增量输出按约 200 ms flush；
- 单条最终输出有约 1,000 token 的 head+tail 预算；
- completion 后发送 handoff complete ack。

voice session 可以跨零个、一个或多个 backing turn，但每个实际编码任务仍由主 `Session` 执行。

#### k. 生命周期和队列防止“声画不同步”

Realtime manager 同一时刻只保留一个 active conversation；start 新会话会先停止旧会话：

```text
realtime_active = false
stop_token.cancel()
await input task
stop/detach fanout task
```

输入队列有类别上限：

- audio 256；
- text 64；
- handoff output 64；
- output events 256。

audio 队列满时丢帧并继续，因为语音实时性优于无限积压；text/handoff 则走有界的异步通道。
`RealtimeHandoffAdmission` 用 semaphore 串行 handoff 与“安全 failure 后 retire”，防止
已经解析但尚未提交的 handoff 在 misalignment 之后启动新 turn。

### ④ 取舍

**Code Mode 省掉的是模型轮次，不是执行轮次。**

一次 JS cell 可以发起几十个 nested calls，显著减少“模型看结果 → 再决定下一调用”的采样次数，
也省掉中间结果在 prompt 中反复出现。代价是调试面更大：脚本错误、yield、cell 取消、
nested 审批和 tool result 顺序都要同时正确。

**为什么 nested calls 不能绕过普通工具链？**

如果 V8 能直接调用文件系统或网络，Code Mode 就会成为权限模型的旁路，审批、sandbox、
hooks、rollout 和预算全部失效。当前设计让 V8 只产生“调用意图”，host 再按普通 tool
routing 执行。

代价是每次 nested tool 都有跨语言/跨进程 callback，比纯 JS 函数调用重；但可审计性来自这层边界。

**为什么 CodeModeOnly 要 fail closed？**

`CodeModeOnly` 的模型 prompt 已经承诺“工具只通过 exec/wait 暴露”。如果 host 不可用却
悄悄切 Direct，模型会看到一套与指令不一致的工具表，破坏缓存与行为预期。普通 CodeMode
可以 fallback Direct，因为它仍允许直接工具；Only 模式没有这个退路。

**Realtime 为什么不是主循环的替代品？**

语音模型擅长低延迟对话和意图确认，不适合承载完整代码上下文、几百次工具调用和严格审批。
复杂工作通过 handoff 进入主 agent，再由主 agent 的 turn loop 执行；Realtime 只负责
交互、播报和必要的 transcript。

代价是一次语音请求可能跨多个 backing turns，UI 必须处理 transcript 与 turn 的交错，
也要限制 startup context 和 handoff payload，避免把主上下文原样复制到低延迟通道。

**为什么 WebRTC 仍需要 WebSocket sideband？**

WebRTC 负责音频媒体，但不适合作为所有结构化控制/事件的主通道。sideband WebSocket 负责
会话初始化、realtime events、handoff 和文本消息，WebRTC 只处理低延迟 audio。
这把“媒体传输”和“agent 控制”拆开，也让 direct WebSocket、WebRTC 和 existing call
共享大部分 `RealtimeConversationManager` 逻辑。

回到收束：§16 完成后，Codex 的执行面已经有三种入口——标准 Responses tool loop、
Code Mode 的程序化 cell、Realtime 的语音/委派通道。下一部分把四幕重新压成一张
架构总览，并给出新增能力的落地清单。

---

## 幕末收获

读完这一幕，你应该能：

- 解释长会话下上下文如何"再生"而不是简单截断
- 说清多智能体的拓扑、通信与控制面
- 区分 summary compaction、remote v2 和 token-budget reset
- 判断一个任务应该放在 root、child agent、Code Mode 还是 Realtime handoff 中
- 解释语音 handoff 为什么仍回到原来的 `Session`，而不是另起一套主循环

---

## 全系列收束

### 一页架构总览

把四幕接回一条线，Codex 的主链路可以压缩成：

```text
用户 / TUI / app-server client
  │
  ▼
app-server
  ├─ thread/start / resume / fork
  ├─ turn/start，解析输入与 client metadata
  └─ 选择 thread_extensions()
       │
       ▼
ThreadManager
  └─ CodexThread / LiveThread
       ├─ 规范 history + rollout
       └─ Session
            │
            ├─ 准入与 turn 生命周期
            ├─ 扩展生命周期 contributor
            ├─ 上下文组装
            │    ├─ base instructions
            │    ├─ history
            │    ├─ ContextContributor
            │    └─ WorldState diff
            ├─ 工具表捕获
            │    ├─ core tools
            │    ├─ ToolContributor
            │    ├─ MCP / plugins / Apps
            │    ├─ deferred tool_search
            │    └─ Code Mode exec / wait
            ├─ 模型请求
            └─ 工具循环
                 ├─ approval / execpolicy / sandbox
                 ├─ ToolExecutor
                 └─ 结果按 call 顺序回灌
```

把横切能力叠上后，可以看成五层：

| 层 | 主要对象 | 负责什么 | 不负责什么 |
|---|---|---|---|
| 进程与协议 | app-server、CodexThread | RPC、线程注册、输入/事件桥接 | 模型推理与工具语义 |
| 会话运行时 | `Session`、`TurnContext`、`StepContext` | turn 生命周期、step 快照、取消 | 具体扩展能力 |
| 上下文 | `ContextManager`、`WorldState`、compaction | 决定模型这一圈看见什么 | 实际执行动作 |
| 工具与控制 | registry、router、approval、Code Mode、agents | 谁能执行、如何并行、何时停止 | 长期知识整理 |
| 持久化与能力扩展 | rollout、thread store、contributors、skills/memory/MCP | 跨进程恢复与能力注入 | 重写主循环 |

三条贯穿全系列的不变量最重要：

1. **模型可见历史只增量变化。** 正常上下文追加，跨 turn 状态用 world-state diff，真需要
   丢弃历史时才用 compaction。
2. **执行必须能归因。** 每次 model/tool/approval/agent 调用都有稳定 id，并绑定到捕获的
   step 配置与权限。
3. **能力可以扩展，主循环不能旁路。** contributor、skill、plugin、MCP 和 Apps 都必须
   回到同一套上下文、工具、审批和持久化边界。

### 扩展开发指南

新增一个能力时，先判断它是哪一种形态：

| 需求 | 首选形态 |
|---|---|
| 增加模型知识、流程、脚本资源 | Skill |
| 增加跨会话经验 | Memory 或 memory extension |
| 接外部工具 / 私有数据 | MCP server，必要时包成 Plugin / App |
| 参与 turn 生命周期、上下文、工具表或审批 | 新 extension crate + Contributor |
| 需要新的主循环阶段或不可替换的执行语义 | 先改 core/extension API，再开放给扩展 |

如果是新的 host 内能力，标准落地顺序是：

```text
1. 在 codex-rs/ext/xxx 建 crate
2. 定义 extension value 与 extension-private state
3. 实现真正需要的最小 contributor 集合
4. 写 install(&mut ExtensionRegistryBuilder<Config>, deps...)
5. 在 app-server/src/extensions.rs 的 thread_extensions() 装配
6. 若暴露 tools，绑定当前 step / environment / permission authority
7. 若有持久状态，设计 on_thread_resume 与 rollout checkpoint
8. 添加入口、生命周期、恢复和失败路径的集成测试
```

几个实现约束：

- **不要先改 `codex-core`。** 新概念优先放进独立 crate；core 只增加真正通用的接点。
- **状态放对层级。** session 依赖放 `session_store`，thread 持久状态放 `thread_store`，
  本轮临时状态放 `turn_store`；extension value 本身跨 thread 共享。
- **不要持有 live 主循环对象。** contributor 接收稳定 id 和受控 input，返回声明式结果。
- **工具必须绑定 step。** 不能拿当前全局配置执行一个由旧 step 发起的调用。
- **所有模型可见输出都要有界。** 新 fragment、tool schema、skill catalog、memory 和
  handoff 都要有 token / byte 上限。
- **恢复是协议。** 只要扩展写入持久状态，就必须说明旧 rollout、版本升级和失败重试时怎样读。
- **配置变化要重投影。** 修改 config shape 时更新 schema；依赖变化时同步锁文件。
- **测试从用户行为出发。** agent 逻辑变化优先加 `core/suite` integration test；app-server
  行为走 JSON-RPC API 测试；UI 变化必须覆盖 snapshot。

可以用一句话判断实现是否合适：

> 去掉这个扩展后，core 的主循环、历史前缀、权限链和恢复语义是否完全不变？

如果答案是否定的，说明扩展越过了 §10 的接点边界。

### 与主流框架的定位对比

这里比较的是**架构取向**，不是最新版本功能表，也不声称各框架内部实现完全相同。

| 维度 | Codex | LangChain 的典型抽象 | LangGraph 的典型抽象 | Dify 的典型抽象 |
|---|---|---|---|---|
| 控制流 | host-owned agent turn loop，模型在固定阶段选择工具 | chain / agent / callback 组合 | 显式图与共享 state，边决定下一步 | 应用/workflow 编排 |
| 上下文 | rollout + 增量 history + world-state diff + compaction | 多由 chain / retriever 组合决定 | graph state 由节点读写 | workflow 变量与节点上下文 |
| 扩展 | 编译期 contributor + 声明式 skill/plugin/MCP/Apps | 组件与集成生态 | 自定义节点/边/state | 可视化节点、模型与工具提供方 |
| 持久化 | thread/rollout 是规范历史，支持 resume/fork | 通常由应用另配 memory/store | checkpointer 等状态持久化抽象 | 平台内会话/工作流状态 |
| 安全边界 | 审批、execpolicy、sandbox、网络代理在核心执行链内 | 多数由应用和工具实现自行组合 | 多数由节点/应用实现组合 | 平台能力与应用配置共同决定 |
| 开放程度 | 主循环封闭，能力插槽开放 | 组合自由度高 | 图拓扑自由度高 | 平台内编排自由度高 |

差异不在于谁“更智能”，而在于把什么当成一等公民：

- LangChain 更强调组件组合；
- LangGraph 更强调显式状态图和可控路由；
- Dify 更强调平台化 workflow 与应用交付；
- Codex 更强调**一个能安全执行代码、可恢复、可扩展的单 agent runtime**。

Codex 的 contributor 设计牺牲了运行期插件自由度，换来稳定生命周期和 prompt-cache
友好的上下文；多智能体也没有开放成任意 swarm，而是 host-owned task tree；Code Mode
提供程序化控制流，但 nested tool 仍回到审批链。这些选择共同服务于一个目标：

> 让模型获得足够强的执行能力，同时让 host 始终能解释、限制、恢复和审计它。

→ 返回 [第一幕：最小闭环](act-01-minimal-loop.md)
