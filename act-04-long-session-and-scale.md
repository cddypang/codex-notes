# 第四幕：长会话与规模化 —— 成熟智能体

> **幕目标**：讲清长会话下的上下文管理、多智能体拓扑，
> 以及主循环之外的两种执行形态。
>
> **前置依赖**：全部前三幕。特别是第一幕第 5 节（world state）——压缩与它强耦合。
>
> **状态**：大纲（待展开正文）

---

## 目录

- [14. Compaction](#14-compaction)
- [15. 多智能体](#15-多智能体)
- [16. Code Mode 与 Realtime](#16-code-mode-与-realtime)
- [幕末收获](#幕末收获)

---

## 14. Compaction

### ① 问题

会话越来越长，上下文窗口迟早会满。直接删掉历史？那模型就失忆了。

### ② 代码入口

```text
codex-rs/core/src/compact.rs
codex-rs/core/src/compact_remote_v2.rs
codex-rs/core/src/compact_remote_history.rs
codex-rs/core/src/compact_token_budget.rs
codex-rs/core/src/compact_model_fallback.rs
codex-rs/core/src/tasks/compact.rs
codex-rs/core/src/session/token_budget.rs
```

### ③ 关键设计

- 三种触发：自动（超阈值）/ 手动（`/compact`）/ mid-turn
- `InitialContextInjection`：`DoNotInject` vs `BeforeLastUserMessage`
  ——为什么 mid-turn 压缩必须把摘要当成最后一条（`compact.rs` 里的注释解释了这个训练假设）
- 摘要 prompt（`SUMMARIZATION_PROMPT` / `SUMMARY_PREFIX`）与替换历史
- 远端压缩（v2）与本地回退
- `PreCompact` / `PostCompact` hook
- **与 world state 的交互**：`ContextContributor::retain_world_state_after_compaction()`
  ——压缩丢掉了渲染文本，但扩展可以保留自己的 section 元数据
- `rollout_budget` / `TokenBudgetContext` / 上下文窗口提示

### ④ 取舍

- 压缩是不可逆的信息损失，怎么界定"什么该活下来"
- 为什么 pre-turn 和 mid-turn 的策略必须不同
- 远端压缩 vs 本地压缩：一致性、成本、可控性

---

## 15. 多智能体

### ① 问题

一个 agent 干不完（或者不该干）所有事，怎么分工？

### ② 代码入口

```text
codex-rs/core/src/thread_manager.rs              # spawn_legacy_subagent
codex-rs/core/src/agent/{mod,control,role,registry,agent_resolver,child_config}.rs
codex-rs/core/src/tools/handlers/multi_agents*.rs
codex-rs/ext/agent/src/lib.rs
codex-rs/agent-roles/src/lib.rs
codex-rs/agent-graph-store/src/lib.rs
codex-rs/agent-message-board-client/src/lib.rs
codex-rs/core/src/context/inter_agent_message.rs
```

### ③ 关键设计

- thread 树：`root_thread_id` / 父子拓扑 / `AgentGraphStore`
- task-path 路由（`Feature::MultiAgentV2`）与 `MultiAgentVersion`
- agent roles：角色文件加载与解析
- 控制工具：`spawn_agent` / `send_input` / `wait` / `close_agent` / `resume_agent`
- agent 间通信：`InterAgentMessage` / `SubagentNotification` / message board
- 上下文隔离与继承：子 thread 拿到什么、不拿什么
- `AgentInvocation` / `AgentRun`：在 fork 出来的上下文里跑初始 prompt
- feature 开关：`Collab` / `MultiAgentV2` / `AgentMessageBoard`

### ④ 取舍

- 为什么默认关闭（token 成本、调度复杂度、可观测性）
- 多 agent 与"一个大 context"的取舍
- 与 LangGraph 的 supervisor / swarm 模式对比

---

## 16. Code Mode 与 Realtime

### ① 问题

除了"一次一个工具调用"，还有别的执行形态吗？

### ② 代码入口

```text
codex-rs/code-mode/src/lib.rs
codex-rs/code-mode-runtime/src/{lib,runtime,service,v8_init}.rs
codex-rs/tools/src/code_mode.rs
codex-rs/core/src/session/code_mode_warning.rs
codex-rs/core/src/realtime_*.rs
codex-rs/realtime-webrtc/
codex-rs/voice-host/
```

### ③ 关键设计

Code Mode：

- 内嵌 V8（`v8_init.rs`），把工具定义转换成 JS 定义（`tool_spec_to_code_mode_tool_definition`）
- 模型写 JS 调工具，而不是一轮一个工具调用
- 两个入口：`exec` / `wait`
- `CodeModeOnly`：模型可见工具被压缩到只剩两个入口
- 相关 feature：`CodeMode` / `CodeModeHost` / `CodeModePrewarm` / `CodeModeInterrupt`
- 与工具循环的关系：嵌套调用、可观测性、审批如何穿透

Realtime：

- WebRTC 传输与 `realtime-webrtc`
- `RealtimeConversationManager` / `realtime_history` / `realtime_context` / `realtime_prompt`
- `RealtimeDelegation` / `RealtimeStartInstructions`：主会话如何委派给实时会话
- 与主 turn 循环的边界

### ④ 取舍

- Code Mode 换来的效率 vs 可控性（审批、审计、沙箱如何穿透）
- Realtime 的定位：它不是替代主循环，而是另一条通道

---

## 幕末收获

读完这一幕，你应该能：

- 解释长会话下上下文如何"再生"而不是简单截断
- 说清多智能体的拓扑、通信与控制面
- 判断什么场景适合 Code Mode

---

## 全系列收束

本文档末尾计划包含三块：

1. **一页架构总览**——把四幕串成一张图（模型层 / 上下文 / 工具 / 扩展 / 持久化）
2. **扩展开发指南**——照着前面的内容，写出"新增一个能力"的标准步骤：

   ```text
   建 ext/xxx crate
     -> 实现需要的 contributor trait
     -> 写 install()
     -> 在 app-server/src/extensions.rs 的 thread_extensions() 加一行
     -> 补集成测试（core/tests/suite 下的 test_codex）
   ```

3. **与主流框架的定位对比**——与 LangChain / LangGraph / Dify 的差异收束
   （控制流归属、上下文策略、扩展模型、执行安全）

→ 返回 [第一幕：最小闭环](act-01-minimal-loop.md)
