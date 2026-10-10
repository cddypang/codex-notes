# 第二幕：工具循环 —— 让模型能动手

> **幕目标**：讲清工具从"定义"到"执行"到"被拦下"的完整链路。
> 这一幕是编码智能体区别于聊天机器人的地方。
>
> **前置依赖**：第一幕（尤其是第 2 节的 turn 循环、第 4 节的上下文组装）。
>
> **状态**：大纲（待展开正文）

---

## 目录

- [6. 工具抽象](#6-工具抽象)
- [7. 工具循环与并发](#7-工具循环与并发)
- [8. 审批与安全边界](#8-审批与安全边界)
- [幕末收获](#幕末收获)

---

## 6. 工具抽象

### ① 问题

`shell`、`apply_patch`、MCP 工具、code-mode 工具、多智能体控制工具——
这些实现方式完全不同，怎么用一套接口统一？

还有一个更微妙的问题：**有些"工具"根本不由 Codex 执行**。
最典型的是 web search——同一个能力，有服务端执行和本地执行两种归属。
所以这一节要同时回答"怎么统一"和"谁负责执行"。

### ② 代码入口

```text
codex-rs/tools/src/lib.rs                       # 对外导出的工具原语
codex-rs/tools/src/{tool_spec,tool_executor,tool_call,tool_payload}.rs
codex-rs/core/src/tools/registry.rs             # CoreToolRuntime / 注册表装配
codex-rs/core/src/tools/spec_plan.rs            # 从 contributor 收集工具
```

### ③ 关键设计

- `ToolSpec` 的形态（`Function` / `Namespace` / `Freeform`……）——写作时需核实完整枚举
- `ResponsesApiTool` / `ResponsesApiNamespace` / `defer_loading` / `output_schema`
- `ToolExecutor` trait + `ToolExecutorFuture`：异步、可取消
- `ToolCall` / `ToolCallSource` / `ToolPayload`：调用来源会影响策略
- `ToolExposure`：哪些工具对模型可见
- `CoreToolRuntime`（registry.rs）在 `ToolExecutor` 之上加的 core 专属元数据：hooks、telemetry、tool search、参数 diff
- 命名空间设计：为什么不是一层扁平名（`mcp__` 前缀与 `NonPrefixedMcpToolNames`）
- 工具从哪来：内置 + `ToolContributor` + MCP + dynamic tools
- **同一个能力的两种执行归属**（以 web search 为例，详见 `act-01` §3 的能力协商）：
  - hosted：`ToolSpec::WebSearch`（`tools/src/tool_spec.rs:39`）只声明 `{"type":"web_search"}`，
    **服务端执行**；本地没有执行器——`registry.rs:469` 给它的是空描述，`router.rs:193` 返回 `false`
  - standalone：`ext/web-search` 注册的本地工具 `web.run`，
    **Codex 自己**POST `alpha/search`（`codex-api/src/endpoint/search.rs:15`）
  - 两者**互斥**：`spec_plan.rs:633` 的 `!standalone_web_search_available` 保证不会同时出现
  - 判定入口：`standalone_web_search_enabled()`（`spec_plan.rs:1038`），
    受 `Feature::StandaloneWebSearch`、provider 能力、`use_responses_lite` 三者共同约束
  - client 侧对 hosted 检索只做**渲染**：`core/src/web_search.rs` 把 `WebSearchAction`
    转成界面文字

### ④ 取舍

- 为什么工具要带 `output_schema`？结构化输出换来了什么
- `defer_loading` / `tool_search` 的设计动机（工具太多会撑爆上下文）
- 为什么 `ToolExecutor` 的签名里要带 `ToolEnvironment`

---

## 7. 工具循环与并发

### ① 问题

模型一次返回多个工具调用怎么办？循环在哪里终止？工具失败了怎么回灌？

### ② 代码入口

```text
codex-rs/core/src/tools/router.rs               # 调用分发
codex-rs/core/src/tools/orchestrator.rs
codex-rs/core/src/tools/parallel.rs             # 并发与取消
codex-rs/core/src/tools/executed_tool_calls.rs  # 调用记录
codex-rs/core/src/tools/lifecycle.rs            # 生命周期通知
codex-rs/core/src/tools/events.rs
```

### ③ 关键设计

- 从模型输出到 handler 的匹配路径
- 并行执行：并发上限、取消传播（`Feature::CodeModeInterrupt` / `InstantInterrupt`）
- 结果回灌格式：`FunctionCallOutputPayload` / `ResponseInputItem`
- 错误分类：`FunctionCallError::RespondToModel` vs `Fatal`
  （见 `ext/memories/src/tools/mod.rs` 的 `backend_error_to_function_call`）
- 控制类工具 vs 普通工具（`is_builtin_control_tool`）
- 终止条件：没有新的工具调用 → turn 结束

### ④ 取舍

- 并行带来的确定性损失，怎么用事件顺序弥补
- 为什么错误要分成"回给模型"和"崩掉"两类

---

## 8. 审批与安全边界

### ① 问题

模型决定执行 `rm -rf /`、读取 SSH 私钥、或者往生产环境发请求——谁拦？

### ② 代码入口

```text
codex-rs/core/src/tools/approvals.rs
codex-rs/core/src/exec_policy.rs                # execpolicy
codex-rs/execpolicy/
codex-rs/sandboxing/                            # 平台抽象
codex-rs/core/src/guardian/                     # 审查
codex-rs/guardian-context/src/lib.rs            # 可组合的审查上下文
codex-rs/ext/guardian-v2/src/{sync_reviewer,async_scorer}
codex-rs/core/src/tools/network_approval.rs
```

### ③ 关键设计

- `AskForApproval` 策略等级；审批请求的产生与裁决
- `ApprovalReviewContributor::decide()`：**认领语义**（第一个返回 `Some` 的赢）
- execpolicy：命令规则匹配（见 `docs/execpolicy.md`）
- 沙箱层级：Seatbelt / Landlock / bubblewrap / Windows Sandbox——只讲它在流程里的位置，不展开平台实现
- 提权路径：`shell-escalation`、`request_permissions`
- Guardian：同步审查（阻断决策）+ 异步打分（风险评分）；`guardian-context` 的 section 组成与 token 预算；`ContextTarget::{Sync, Async}`
- 网络策略：`network-proxy`、`network_policy_decision.rs`

### ④ 取舍

- 安全 vs 流畅：审批骚扰会让人关掉保护
- 为什么 Guardian 要单独起一次模型调用，而不是让主模型自审
- 与 LangChain / Dify 的对比：为什么通用框架在这一层几乎空白

---

## 幕末收获

读完这一幕，你应该能：

- 说清一个工具调用从"模型输出"到"结果回灌"经过的所有关卡
- 解释审批、execpolicy、沙箱、guardian 四者各自负责哪一段
- 判断一个新工具应该接入在哪一层

## 下一幕引子

现在模型能动手了，但进程一退会话就没了，模型也记不住任何事。
下一幕解决"持久化"，以及"怎么让别人给 core 加能力而不改 core"。

→ [第三幕：记忆与扩展](act-03-memory-and-extensions.md)
