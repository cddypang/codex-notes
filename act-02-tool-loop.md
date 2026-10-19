# 第二幕：工具循环 —— 让模型能动手

> **幕目标**：讲清工具从"定义"到"执行"到"被拦下"的完整链路。
> 这一幕是编码智能体区别于聊天机器人的地方。
>
> **前置依赖**：第一幕（尤其是第 2 节的 turn 循环、第 4 节的上下文组装）。
>
> **状态**：第 6–7 节正文完成；第 8 节仍为大纲

---

## 目录

- [6. 工具抽象](#6-工具抽象)
- [7. 工具循环与并发](#7-工具循环与并发)
- [8. 审批与安全边界](#8-审批与安全边界)
- [幕末收获](#幕末收获)

---

## 6. 工具抽象

### ① 问题：一次请求里为什么会同时出现 shell、MCP 和 web search

假设用户输入：

> 在仓库里找一下 `run_turn` 在哪里处理工具调用；再查一下我明天上午的日历；
> 最后看看这个 API 最新版本的文档。

这不是三个“函数调用”问题，而是三种完全不同的执行世界：

1. 查代码要在当前开发环境里运行命令。模型可能会执行 `rg -n "run_turn"`，
   也可能先 `ls` 看一下目录。真正干活的是 shell 和它启动的子进程。
2. 查日历要调用一个 MCP server。Codex 需要先发现服务器提供的工具，再把结构化参数
   发给另一个进程或远端服务。
3. 查最新文档可能由 Responses API 在服务端完成，也可能由 Codex 自己请求搜索 endpoint。
   两种 web search 对模型看起来相似，执行归属却完全不同。

所以本节不是先给一套抽象名词再找例子，而是反过来：沿着上面三类真实需求，
看 Codex 如何把“模型能看到什么”和“host 实际能执行什么”接起来。

先记住四个概念，后面每个场景都会用到：

| 概念 | 回答的问题 | 典型例子 |
|---|---|---|
| `ToolSpec` | 模型看到的名字、描述和参数 schema 是什么 | `exec_command`、`web.run`、`web_search` |
| `ToolName` | host 如何唯一定位一个工具 | `functions.exec_command`、`web.run`、`mcp__calendar.create_event` |
| `ToolExecutor` | 谁真正执行，以及返回什么 | shell handler、MCP handler、web-search extension |
| `ToolExposure` | 工具是否直接可见、需要搜索，还是只能从 Code Mode 调 | direct shell、deferred MCP、code-mode-only 工具 |

这四个问题经常分离。比如 hosted `web_search` 有 `ToolSpec`，却没有本地 executor；
deferred MCP 工具已经在 registry 里，却不会出现在初始工具表；`tool_search` 能发现工具，
但它本身不负责启动 MCP server。

### ② 代码入口：跟着三类工具走一遍

所有场景的共同起点仍是第一幕的 `Prompt`：

```text
core/src/session/turn.rs:1589   build_prompt
  tools: step_context.tool_router.model_visible_specs()
```

模型只能根据 `Prompt.tools` 里的定义发起调用。这个列表在更早的
`capture_step_context_inner`（`core/src/session/mod.rs:3720`）中生成：

```text
capture_step_context_inner
  ├─ mcp_runtime_for_step                 # 捕获本轮 MCP 连接与工具目录
  └─ turn::built_tools（core/src/session/turn.rs:1800）
       └─ build_tool_router（core/src/tools/spec_plan.rs:123）
            ├─ add_core_tool_sources
            ├─ append_mcp_tools
            ├─ 扩展 ToolContributor
            ├─ dynamic tools
            ├─ hosted tools
            └─ finalize_tool_router
                 └─ ToolRouter + model_visible_specs
```

工具表是**每个采样步骤**重新捕获的；首圈在 `run_turn`
（`core/src/session/turn.rs:259`）准备，后续每圈在
`core/src/session/turn.rs:465` 再捕获。因此同一个 turn 内，MCP 刚启动、配置变化或工具搜索
命中，都可能改变下一次模型请求看到的工具表。

#### 场景 A：查代码时，`grep` 和 `ls` 是工具吗？

不是。至少在主线里，它们不会产生两个叫 `grep_tool`、`ls_tool` 的工具调用。
模型看到的是通用 shell 工具 `exec_command`，然后把命令作为字符串放进去：

```json
{
  "name": "exec_command",
  "arguments": "{\"cmd\":\"rg -n \\\"run_turn\\\" codex-rs/core/src\",\"workdir\":\"D:/Work/03github/codex\"}"
}
```

这条路径在代码里是：

```text
core/src/tools/spec_plan.rs:1079          add_shell_tools
  └─ registry.add(ExecCommandHandler)
       core/src/tools/handlers/unified_exec/exec_command.rs:112   ToolExecutor impl
       └─ :155                                 handle_call
            ├─ core/src/tools/handlers/unified_exec.rs:100   get_command
            │    └─ shell.derive_exec_args(&args.cmd, ...)
            └─ UnifiedExecProcessManager::exec
```

`create_exec_command_tool_with_environment_id`（`core/src/tools/handlers/shell_spec.rs:24`）
给模型的参数不是“文件名”或“搜索正则”，而是：

- `cmd`：整个 shell command；
- `workdir`：在哪个目录执行；
- `tty`：是否分配 PTY；
- `yield_time_ms`：长命令多久后返回 session id；
- `max_output_tokens`：输出预算；
- 多环境时还可选 `environment_id`；
- 必要时还有 sandbox/升级权限字段。

因此 `rg`、`grep`、`ls`、`cat`、`git`、`cargo` 以及用户自定义命令都共享同一个工具边界。
`get_command` 最终调用 `Shell::derive_exec_args`（`core/src/shell.rs:22`），把 command 字符串交给
当前环境的 shell。`grep` 能不能用取决于目标环境是否安装了这个二进制；Windows 上模型更常会写
`Get-ChildItem`、`Select-String` 或 PowerShell 等价命令。Codex 不把 Unix 命令名写死成工具，
而是把 OS 和 shell 差异留给命令本身与模型 guidance。

实际使用中，`exec_command` 覆盖了编码任务的大半“手工动作”：查文件、看 `git status/diff`、
运行测试和构建、启动开发服务器、执行仓库脚本。短命令在 `yield_time_ms` 内结束就直接返回输出；
长命令可以返回 `session_id`，再通过 `write_stdin` 继续读输出或写入 stdin。若 UnifiedExec 被关闭，
handler 改成 one-shot spec：同名工具仍能执行命令，但移除 `tty`/`yield_time_ms`，改用
`timeout_ms`，命令不能跨调用续接。

这和 `apply_patch` 形成很直观的对照。`apply_patch` 在模型看到的工具表里是一个独立的
Freeform 工具（`core/src/tools/handlers/apply_patch_spec.rs:9`；仅在模型/配置启用时注册），
输入是 patch 文本，
不是 JSON 参数。也就是说：

- “搜索代码”通常走 `exec_command`；
- “按 patch 语义修改文件”可以直接走 `apply_patch`；
- shell 仍然能执行 `apply_patch` CLI，但那属于 shell 命令路径，不是工具表里的独立调用。

`exec_command` 返回的是 `ExecCommandToolOutput`，最终变成 `function_call_output`。
它可能包含 `output`、`exit_code`、`wall_time_seconds`、`session_id` 和截断信息。
模型看到搜索结果后，可以在同一个 turn 里继续调用 `rg` 缩小范围，或调用 `apply_patch` 修改文件。
这就是第一幕所说“turn 级循环”的实际来源。

#### 场景 B：MCP 查日历，server 什么时候启动？

假设配置里有一个本地 stdio MCP server：

```toml
[mcp_servers.calendar]
command = "/path/to/calendar-mcp"
args = ["--stdio"]
startup_timeout_sec = 30
```

它的生命周期通常不是“模型第一次调用工具时才 `spawn`”。主线是：

**1. 创建 Session 时安装 MCP runtime**

`Session` 构造过程中，`core/src/session/session.rs:1891` 调用
`install_initial_mcp_runtime`（`core/src/session/mcp_runtime.rs:107`）。
后者构造 `McpRuntimeInput`，交给 `McpRuntime::replace`，再进入
`McpConnectionSet::new`（`codex-rs/codex-mcp/src/connection_manager.rs:230`）。
配置中的 MCP server 在这一阶段被 reconcile，stdio server 开始启动并做 MCP 初始化、列工具。

主 session 默认使用 `McpStartupPolicy::Eager`
（`codex-rs/codex-mcp/src/runtime.rs:71`）；子 agent 在有上一份 runtime 时使用
`LazyWhenCached`。所以“配置里写了 server”与“工具此刻一定能用”仍是两件事：
进程可能正在握手，也可能因为认证、超时或命令不存在而失败。

**2. 初始化并不等于等所有 server 都 ready**

- `required = true` 的 server 由 `validate_required_servers`
  （`codex-rs/codex-mcp/src/connection_manager/required.rs:17`）把关。默认必须拿到 live connection；
  若 `startup_readiness = "catalog"` 且有有效工具缓存，也可用缓存满足启动条件。
- 普通可选 server 可以继续在后台启动。`capture_binding_with_metadata` 会优先使用缓存目录，
  必要时等待一段共享的 optional startup grace。
- 因此第一次模型请求可能已经看到某个 MCP 工具，也可能暂时看不到；server 稍后 ready，
  下一次 `capture_step_context` 重新捕获工具表时才出现。

**3. 用户输入可以提前抬高启动要求**

`required_mcp_servers_for_input`（`core/src/session/turn.rs:930`）会在首圈采样前扫描：

- 显式提到的 plugin；
- `mcp://server/path` 形式的资源引用；
- skill 依赖的 MCP server；
- 可解析的 connector/app 提及。

这些名字被传给 `capture_step_context_with_required_mcp_servers`，最终进入
`mcp_runtime_for_step`（`core/src/session/mcp.rs:363`）和
`current_binding_with_requirements`（`codex-rs/codex-mcp/src/runtime.rs:418`）。
这一步会刷新 dirty runtime，并等待本轮明确依赖的 server。这样用户说“用 @calendar 查一下”
时，不会因为普通 MCP 还在后台启动而随机拿不到日历工具。

**4. 工具表来自该 step 的 MCP binding**

server 初始化并列出工具后，`McpBinding.tools()` 提供 `ToolInfo`，再由
`McpHandlerCache::append_mcp_tools`（`core/src/mcp_tool_exposure.rs:38`）注册成
`McpHandler`（`core/src/tools/handlers/mcp.rs:139`）。模型侧的 namespace 通常是
`mcp__calendar`，raw MCP name 仍保留在 `ToolInfo.tool.name`。

如果模型支持 tool search，普通 MCP 工具通常注册为 `Deferred`。第一轮请求里看到的不是
日历工具本身，而是 `tool_search`：

```json
{"name":"tool_search","arguments":"{\"query\":\"calendar list events\",\"limit\":5}"}
```

`tool_search` 在本地 BM25 索引里搜索已经捕获的 MCP 工具，返回 `LoadableToolSpec`。
下一次模型请求才看到类似：

```json
{
  "name": "list_events",
  "namespace": "mcp__calendar",
  "arguments": "{\"date\":\"2026-10-20\"}"
}
```

关键结论：**`tool_search` 不负责启动 MCP server**。它只是从当前 `McpBinding` 已经加载的目录里做检索。
如果 server 没启动、也没缓存，它的工具根本不在这份目录里，搜索自然搜不到。

真正执行 MCP 调用时，`McpHandler::handle_call`
（`core/src/tools/handlers/mcp.rs:196`）调用 `Session::prepare_mcp_call`，后者再走
`McpRuntime::prepare_call`（`codex-rs/codex-mcp/src/runtime.rs:543`），拿到当前 runtime 中匹配的
server/client 和调用策略。MCP 进程/连接是按 thread runtime 复用的，不会每个 tool call 重启一次。
线程关闭时才统一 shutdown。

#### 场景 C：搜索最新文档，为什么可能是两种工具

模型想联网时，工具表里可能出现：

- `web_search`：hosted Responses tool；
- `web.run`：Codex 本地 extension 工具。

hosted 形态由 `hosted_model_tool_specs`（`core/src/tools/spec_plan.rs:622`）生成
`ToolSpec::WebSearch`。模型触发后，后端返回 `ResponseItem::WebSearchCall`。
`ToolRouter::build_tool_call`（`core/src/tools/router.rs:248`）不把 `WebSearchCall` 识别成本地
tool call；它作为 response item 直接记录和展示，不需要本地工具结果回灌。

standalone 形态由 `ext/web-search` 的 `WebSearchTool`
（`codex-rs/ext/web-search/src/tool.rs:54`）提供。它声明 `web.run`，有本地 executor，
自己构造搜索请求并把结果转换成 `ToolOutput`。对 `ToolRouter` 来说，它和 shell、MCP 一样是普通工具。

两者何时互斥也应当说得非常具体：只有**内置 standalone search extension 成功注册了 `web.run`**
时，hosted `web_search` 才会被抑制。若一个同名 `web.run` 来自 MCP 或 dynamic tool，
它只是另一个普通工具，当前代码允许它和 hosted `web_search` 同时存在。

#### 三类场景合起来，代码入口可以这样读

```text
【Session 初始化】
core/src/session/session.rs:1891              install_initial_mcp_runtime
codex-rs/codex-mcp/src/runtime.rs:305        McpRuntime::new
codex-rs/codex-mcp/src/connection_manager.rs:230  McpConnectionSet::new

【每个模型请求前】
core/src/session/mod.rs:3720                 capture_step_context_inner
core/src/session/mcp.rs:363                  mcp_runtime_for_step
core/src/session/turn.rs:1800                built_tools
core/src/tools/spec_plan.rs:123              build_tool_router
core/src/tools/spec_plan.rs:349              finalize_tool_router
core/src/session/turn.rs:1589                build_prompt

【模型调用后，第 7 节继续】
core/src/tools/router.rs:248                 build_tool_call
```

### ③ 关键设计：场景背后的通用机制

#### a. `ToolSpec` 是请求侧的“菜单”

`ToolSpec`（`codex-rs/tools/src/tool_spec.rs:22`）直接序列化成 Responses API 的工具声明。
当前有五种 wire 形态：

| 变体 | wire `type` | 现实中的例子 |
|---|---|---|
| `Function` | `function` | `exec_command`、顶层 dynamic function |
| `Namespace` | `namespace` | `web.run`、MCP 工具、多智能体工具、Code Mode 分组 |
| `ToolSearch` | `tool_search` | 搜索 deferred MCP/plugin 工具 |
| `WebSearch` | `web_search` | 服务端执行的 hosted web search |
| `Freeform` | `custom` | `apply_patch` 的 patch 文本 |

`Function` 和 namespace 中的 function 都由 `ResponsesApiTool`
（`codex-rs/tools/src/responses_api.rs:33`）描述。`ToolDefinition` 是其更中性的内部前置形态，
MCP 和 dynamic tool 先使用它解析 name/description/input schema/output schema，再转换成 wire shape。

`ResponsesApiTool.output_schema` 带 `#[serde(skip)]`
（`codex-rs/tools/src/responses_api.rs:44`），不会出现在普通 function tool 声明里。
它主要服务 Code Mode。例如 `exec_command` 的输出 schema 定义在
`shell_spec.rs:198`，包含 `output`、`exit_code`、`session_id` 等字段；Code Mode 可以把嵌套调用
渲染成有返回类型的 TypeScript 声明，而普通 direct tool call 只把这些值当作下一次模型输入的普通结果。

#### b. `ToolName` 才是 registry 的身份

模型调用里的 `name` 和 `namespace` 会先转成
`ToolName { name, namespace }`（`codex-rs/protocol/src/tool_name.rs:12`）。
没有 namespace 的 `exec_command` 在内部规范为 `functions.exec_command`；
日历工具可能是 `mcp__calendar.list_events`。registry 按 `ToolName` 查找，而不是按描述字符串或者
扁平化后的拼接名。

可读模型名和底层协议名故意分离。MCP 层会清洗模型可见名称、处理重名哈希，并限制 API 长度；
但真正发给 MCP server 的 raw server/tool name 留在 `ToolInfo`。同一套机制也支持
`NonPrefixedMcpToolNames`：它只改变模型侧是否保留 `mcp__` 前缀，不改变内部 namespace/name 二元身份。

#### c. `ToolExecutor` 同时拥有 schema 和执行逻辑

统一 runtime contract 是：

```rust
pub trait ToolExecutor<Invocation>: Send + Sync {
    fn tool_name(&self) -> ToolName;
    fn spec(&self) -> ToolSpec;
    fn exposure(&self) -> ToolExposure;
    fn search_info(&self) -> Option<ToolSearchInfo>;
    fn supports_parallel_tool_calls(&self) -> bool;

    fn handle<'a>(&'a self, invocation: Invocation) -> ToolExecutorFuture<'a>
    where
        Invocation: 'a;
}
```

这正好对应前面三个例子：

- `ExecCommandHandler` 的 `spec()` 返回 shell schema，`handle()` 启动命令；
- `McpHandler` 的 `spec()` 返回某个 MCP tool schema，`handle()` 把参数转发给 server；
- `WebSearchTool` 的 `spec()` 返回 `web.run`，`handle()` 发搜索请求。

内置工具拿完整的 `ToolInvocation`（`core/src/tools/context.rs:70`），其中包含 `Session`、
`StepContext`、取消 token、diff tracker 等 core 能力。扩展工具拿公开的 `ToolCall`
（`codex-rs/tools/src/tool_call.rs:132`）；`ExtensionToolAdapter`
（`core/src/tools/handlers/extension_tools.rs:44`）把 core invocation 适配成这个公共形状。

`ToolCall` 里最值得结合场景看的是三项：

- `source: Direct | CodeMode`：是模型直接调 shell，还是由一段 Code Mode 脚本嵌套调用；
- `environments: Vec<ToolEnvironment>`：当前 turn 可选的本地/远端执行环境，不只假设进程 cwd；
- `payload: ToolPayload`：`Function { arguments }`、`ToolSearch { arguments }` 或
  `Custom { input }`，分别对应 JSON 函数、搜索和 `apply_patch` 这类自由文本工具。

执行结果统一实现 `ToolOutput`（`codex-rs/tools/src/tool_output.rs:12`），所以 shell 输出、MCP
`CallToolResult`、搜索 turn item 和 tool-search 结果都能转换成下一次请求需要的
`ResponseInputItem`，同时保留日志、截断、外部上下文和 hook 所需信息。

#### d. `ToolExposure` 决定“什么时候能看到”

工具存在、工具可见、工具可直接调用不是同一件事：

| 值 | 初始工具表 | tool search | Code Mode 嵌套调用 |
|---|---:|---:|---:|
| `Direct` | ✅ | — | ✅ |
| `Deferred` | — | ✅ | ✅ |
| `DeferredModelOnly` | — | ✅ | — |
| `DirectModelOnly` | ✅ | — | — |
| `CodeModeOnly` | — | — | ✅ |
| `Hidden` | — | — | — |

结合真实场景：

- shell 通常是 `Direct`，因为几乎每个编码任务都可能需要；
- MCP 工具在开启 search 时默认 `Deferred`，避免几十个服务端工具把初始上下文撑满；
- 某些控制工具只允许模型直接调用，不进 Code Mode，使用 `DirectModelOnly`；
- 一些内部能力注册但不可见，使用 `Hidden`。

`ToolExposures` 位集合用于把 executor 默认 surface 与 MCP 的 `omit_tools_from` 等策略求交集。
deferred 工具会进入 BM25 目录，`tool_search` 命中后返回 `LoadableToolSpec`；这个 spec 去掉不必要的
output schema 并标记 `defer_loading`，供下一轮请求加载。

#### e. 五类来源最终汇入同一个 registry

`build_tool_router` 的顺序如下：

| 来源 | 谁提供 | 典型例子 | 本地 executor |
|---|---|---|---:|
| core 内置 | `add_core_tool_sources` | `exec_command`、`apply_patch`、view image | ✅ |
| MCP | `append_mcp_tools` | 日历、工单、浏览器连接器 | ✅ |
| 扩展 | `ToolContributor` | memory、skill、`web.run`、image generation | ✅ |
| dynamic | `DynamicToolSpec` | host/client 动态注册的 function | ✅ 回调 host |
| hosted | `hosted_model_tool_specs` | 服务端 `web_search` | ❌ |

core 内置工具按 feature、模型、environment 和 session 类型条件注册。没有可用 environment 就不会有
shell；模型不支持某种 tool mode，就不会出现对应入口。MCP 工具先经过 server filter、connector policy
和名称规范化，再按 search 是否开启决定 direct/deferred。扩展工具通过 `ToolContributor`
（`codex-rs/ext/extension-api/src/contributors.rs:355`）每步返回 executor；dynamic tool 则由
`DynamicToolHandler` 把调用交回 host/client。

trusted 来源内部重名视为实现错误；external 来源重名时跳过后来者并记录 collision。
`finalize_tool_router` 还会处理 namespace 冲突、Code Mode 入口覆盖、tool-search 入口、MCP 曝光收紧
和最终 `model_visible_specs`。

#### f. Code Mode 是“把工具组合起来”的另一个调用面

普通 direct 模式下，一次查代码可能需要：

```text
模型 → exec_command(rg ...) → 模型 → exec_command(cat ...) → 模型 → apply_patch(...)
```

Code Mode 可以让模型一次写一段脚本：

```ts
const hits = await tools.exec_command({ cmd: "rg -n run_turn codex-rs/core/src" });
const files = hits.output;
text(files);
```

此时模型直接调用的是 `exec`，嵌套调用仍会解析到原始 `exec_command` 的 `ToolName` 和 executor，
只是 `ToolCallSource` 变成 `CodeMode { cell_id, runtime_tool_call_id }`。`exec` 和 `wait` 不是新的业务
能力，而是 `register_code_mode_executors`（`core/src/tools/spec_plan.rs:819`）在已有工具表上生成的统一入口。

#### g. 为什么工具表必须跟 StepContext 一起冻结

一次请求发出后，MCP 可能刚好完成启动，配置可能被刷新，或者用户中途发来新输入。如果 host 在调用返回时
重新按“最新工具表”解析，就会出现模型看到 A、host 查 B 的错位。

Codex 的做法是让一次采样步骤使用自己的 `StepContext`：

- `build_prompt` 从它的 `ToolRouter` 取工具表；
- 响应解析从同一个 `ToolRouter` 取身份和显式入口；
- 执行携带同一份 model/settings/environment 快照；MCP 调用则用该步 advertised `ToolInfo`
  在当前 thread runtime 中准备 client 和调用策略。

下一圈才重新捕获。工具表因此能在 turn 内更新，但每次模型请求与它对应的工具调用始终是同一份计划。

### ④ 取舍

**为什么 `grep`、`ls` 不各自做成工具。** 如果每个常用命令都独占一个工具，工具表、schema 和平台分支
会迅速膨胀，用户自定义命令也无法覆盖。统一 shell 换来了最大灵活性，但把风险集中到一个入口：
命令解析、审批、沙箱、输出截断都必须在 `exec_command` 周围做厚。具体的权限处理留到 §8。

**为什么 `apply_patch` 又单独成工具。** 搜索和读文件天然适合 shell；结构化编辑则希望模型输出有明确
语法、可验证、只应用一个 patch。Freeform `apply_patch` 把“修改意图”从任意命令字符串里拿出来，
便于解析、审批和产生 diff。代价是模型需要区分“shell 改文件”和“专用 patch 工具”。

**为什么 MCP server 在 Session 初始化时就启动。** 等第一次调用才拉起进程，会让第一个依赖 MCP 的请求
额外等待握手和列工具，而且无法提前发现认证/配置错误。提前启动和缓存目录把延迟移到会话启动阶段；
代价是未使用的 server 也可能占资源，所以才有 `required`、`enabled`、cached catalog、optional grace
和子 agent 的 lazy policy。

**为什么 `tool_search` 不负责启动 server。** 把发现和执行分开后，搜索结果只是模型面的一份工具声明，
不会隐式产生副作用。server 生命周期由 thread runtime 管理，调用只复用已有连接。代价是 server 没启动
或没缓存时，搜索无法“凭名字”变出工具；用户显式提及 plugin/skill 依赖时，必须在 step 捕获前先等待。

**为什么 direct 与 deferred 并存。** shell 这类高频工具直接出现在初始工具表里，模型可以直接发起调用，
不必先调用 `tool_search`、再等待下一次模型请求加载工具定义；MCP/plugin 数量多、单个 schema 又大，
全部 direct 会挤占上下文。deferred 把成本换成一次搜索和更多检索质量依赖；direct 把成本换成每轮更大的
工具表。让接入点声明默认 surface，再由 host 按配置收紧，是两者之间的折中。

**为什么同一能力允许 hosted 和 local 两种 web search。** 有的 provider 能直接执行搜索，有的环境需要
Codex 自己请求 endpoint。`ToolSpec` 负责向模型表达能力，executor 决定谁执行；hosted 版没有本地
runtime，local 版则发送普通 function call。两者共享“搜索”语义，但不能共享执行路径。

读完 §6，应该能回答：

- 用户让 Codex 查代码时，`rg/grep/ls` 为什么都出现在 `exec_command` 的参数里；
- MCP server 是在什么时候启动、什么时候等待、什么时候会出现在工具表中；
- `tool_search` 命中和 MCP server 启动分别发生在哪一步；
- hosted `web_search` 与本地 `web.run` 为什么一个没有本地 executor；
- 为什么每个 step 都重新组装工具表，却要保证一次请求内的身份和快照一致。

如果模型没有返回工具调用，控制流仍按第一幕的 turn 收尾；如果它返回了 function/custom tool call，
就从 `handle_output_item_done → ToolRouter::build_tool_call` 进入下一节，开始讨论调用如何排队、
并行执行并把结果回灌给模型。

---

## 7. 工具循环与并发

### ① 问题：模型一次返回多个工具调用时，谁先跑、结果怎么排

继续上一幕的代码搜索场景。用户说：

> 修复退款流程的失败测试：分别在 orders 和 billing 模块里搜索 refund 的实现，检查测试失败日志，
> 如果发现明显的断言错误就修掉。

模型第一轮可能返回三个互相独立的调用：

```text
[1] exec_command   rg -n "refund" crates/orders
[2] exec_command   rg -n "refund" crates/billing
[3] exec_command   rg -n "AssertionError|expected|actual" artifacts/refund-test.log
```

注意这里没有 `apply_patch`。模型还没有看到搜索结果和测试日志，无法知道应该修改哪个文件、哪条断言；
修文件依赖前三个调用的结果，必须等它们回灌后，在下一轮模型请求里再决定。

所以这一批调用需要满足几个看似简单、实际互相冲突的要求：

- 两个代码搜索和一次失败日志检索可以同时跑，否则每多一个只读操作就多一次等待；
- 模型收到的工具结果必须按它发出调用的顺序排列，否则 `call_1` 的输出可能被配到
  `call_2` 的上下文位置；
- 用户中断、Code Mode 取消或进程退出时，不能留下“调用已记录但永远没有结果”的悬挂状态；
- 工具错误有时应该交给模型纠正，有时代表 host 已经无法继续。

等三个结果都回灌后，模型才可能在下一轮调用 `apply_patch`。它在并发门中默认独占执行，
不会和上一轮仍在进行的读工具交错。本节沿着这次“并行调查 → 串行修改”的过程，
回答上面的问题。

### ② 代码入口

接点在第一幕第 2 节的 turn 循环和第 6 节末端：

```text
core/src/session/turn.rs:2525              try_run_sampling_request
  └─ ResponseEvent::OutputItemDone
       └─ core/src/stream_events_utils.rs:315   handle_output_item_done
            └─ core/src/tools/router.rs:248     build_tool_call
                 └─ core/src/tools/parallel.rs:77   handle_tool_call

ResponseEvent::Completed
  └─ core/src/session/turn.rs:2472         drain_in_flight
       └─ FuturesOrdered
            └─ core/src/tools/parallel.rs:125   handle_tool_call_with_source
                 └─ core/src/tools/registry.rs:527   dispatch_any_with_state
                      └─ ToolExecutor::handle
                           └─ AnyToolResult::into_response
```

按时间顺序展开：

```text
1. 模型流里出现一个完整的 function/custom tool call
2. build_tool_call 把 ResponseItem 解析成 ToolCall
3. 原始 call item 立刻写入历史 / rollout
4. handle_tool_call 生成一个 future，放入 FuturesOrdered
5. needs_follow_up = true
6. 流结束后 drain_in_flight 开始轮询所有 future
7. 每个 ToolCallRuntime：
     记录调用 → 等 readiness → 进并发门 → registry 分派 → handler
8. ToolOutput 转成 function_call_output / custom_tool_call_output
9. 按模型调用顺序写入历史
10. try_run_sampling_request 返回 needs_follow_up，外层 run_turn 发起下一次模型请求
```

一个容易误解的时序是：**工具 future 会先排队，但不会在模型还没把响应流读完时立即执行。**
`handle_output_item_done` 只把 future 放进 `in_flight`（`core/src/session/turn.rs:2787`）；
直到收到 `ResponseEvent::Completed` 并在流循环结束后，代码才调用
`drain_in_flight`（`core/src/session/turn.rs:3162`）开始轮询它们。代码注释也明确写了：

> The sampling loop collects results in order only after its stream ends.

这样 host 可以先完整记录模型要求调用哪些工具，再统一开始执行，避免模型仍在追加调用时，
前面已经有副作用工具开始修改工作区。

### ③ 关键设计

#### a. 从模型 item 到 `ToolCall`

`ToolRouter::build_tool_call`（`core/src/tools/router.rs:248`）只接受三类可执行调用：

- `ResponseItem::FunctionCall` → `ToolPayload::Function`；
- execution 为 `client` 的 `ToolSearchCall` → `ToolPayload::ToolSearch`；
- `CustomToolCall` → `ToolPayload::Custom`。

`WebSearchCall` 不在这里处理：它可能由服务端执行，最终作为非工具 response item 记录。
这也是第 6 节“有 spec、没有本地 executor”的落点。

成功解析后，`handle_output_item_done`
（`core/src/stream_events_utils.rs:315`）先做一件对恢复很重要的事：

```rust
record_completed_response_item(...).await;
```

工具调用 item 在真正执行前已经进入 thread history 和 rollout。这里记录的是**调用意图**，
不是结果。若进程在工具运行中崩溃，恢复后至少能看到“这个 `call_id` 曾经被请求”，不会凭空丢掉半条调用记录。

随后它创建 `InFlightFuture` 并把 `needs_follow_up` 设为 `true`。这意味着只要模型请求了工具，
本轮模型采样就不可能直接结束；即使执行结果只是一句错误，也要回灌后再让模型决定下一步。

#### b. `ToolCallRuntime`：统一包装一次调用

`ToolCallRuntime`（`core/src/tools/parallel.rs:45`）保存跨调用共享的运行时状态：

- 当前 `Session`；
- 发出调用时对应的 `StepContext` 和 `ToolRouter`；
- 同一个 turn 的 diff tracker；
- 一个 `RwLock<()>`，作为工具并发门。

`handle_tool_call`（`core/src/tools/parallel.rs:77`）先准备 executed-call 记录，再调用
`handle_tool_call_with_source`。后者完成以下顺序：

```text
记录 ToolCall
  → 发出 on_tool_dispatch
  → 等待工具 readiness（例如 MCP server）
  → 获取并行读锁或独占写锁
  → ToolRouter 定位 runtime
  → CoreToolRuntime::dispatch
       → pre-tool hooks / 参数重写
       → notify_tool_start
       → ToolExecutor::handle
       → ToolOutput 转换
       → post-tool hooks / result feedback
       → notify_tool_finish
```

这里的 `dispatch` 是工具执行流水线，不是模型 turn 循环。它内部还可能调用
`ToolOrchestrator`（`core/src/tools/orchestrator.rs:125`）完成“审批 → 选 sandbox → 尝试 →
必要时升级 sandbox 重试”。`exec_command`、`apply_patch` 等进程工具的 handler 会走这条路径；
MCP 或纯内存工具不一定使用同一个 orchestrator。

关键点是：**orchestrator 的一次 sandbox 重试仍属于同一个 `ToolCall`**。它不会重新进入模型循环，
也不会生成第二个 function call；模型最终只看到这次调用的一份最终输出。审批和安全策略的细节留到 §8。

#### c. 并发规则：read 锁并行，write 锁独占

是否允许并行由工具 runtime 声明：

```text
router.tool_supports_parallel(call)
  → Registry::supports_parallel_tool_calls(tool_name)
  → ToolExecutor::supports_parallel_tool_calls()
```

`supports_parallel == true` 时拿 `RwLock` 的 read guard，多个调用可以同时运行；
否则拿 write guard，和所有其它工具互斥。当前典型取值：

| 工具 | 并行 | 原因 |
|---|---:|---|
| `exec_command` | ✅ | 允许同时发起多个独立命令 |
| `write_stdin` | ✅ | 可以并行轮询/写入不同 terminal session |
| `view_image` | ✅ | 只读 |
| `web.run` | ✅ | 搜索调用独立 |
| `tool_search` | ✅ | 纯本地目录检索 |
| 默认 MCP 工具 | ❌ | 除非 server 显式 opt-in |
| MCP `readOnlyHint` 工具 | ✅ | server 声明只读时可以并行 |
| `apply_patch` | ❌ | 修改共享工作区，需要独占 |

注意这里没有另设一个“最多同时 N 个工具”的 semaphore。实际的 fan-out 上限就是
模型在一个响应里发出的、可并行工具的调用数量；非并行调用则通过读写锁形成顺序屏障。

`exec_command` 声明支持并行，并不代表它能证明每条 shell command 都是只读的。并发门是
**工具级声明**，不是逐条命令分析；`cargo test`、`git status`、文件生成脚本之间的冲突仍要靠
模型选择、approval、sandbox 和命令本身的行为约束收敛。

#### d. `FuturesOrdered`：并行执行，但按调用顺序回灌

`in_flight` 的类型是 `FuturesOrdered<InFlightFuture>`（`core/src/session/turn.rs:2589`）。
它会轮询所有未完成 future，但只按**加入顺序**向下游产出结果。

设模型依次返回 A、B、C：

| 时间 | 事件 | 模型历史 |
|---|---|---|
| t0 | A、B 开始 | — |
| t1 | B 完成，实际结果 ready | 等待 A |
| t2 | A 完成 | 先记录 A 的输出 |
| t3 | B 已缓存，立刻取出 | 再记录 B 的输出 |

因此执行完成事件可以是 `B → A`，但模型下一轮看到的工具结果一定是 `A → B`。
`call_trace::result_ready` 在 handler 真正完成时立即记录（`core/src/tools/parallel.rs:233`），
用于观察真实延迟；模型上下文则依赖 `FuturesOrdered` 保持稳定顺序。

如果模型显式返回 `cargo test` 和 `rg` 两个 `exec_command`，它们也可以在响应流结束后同时进入
各自的子进程：`cargo test` 不会“触发”后面的 `rg`，`rg` 也不是在轮询测试状态；它只是另一个独立的
read-lock 工具调用，开始扫描文件。若 `rg` 先完成、`cargo test` 后完成，`rg` 的输出会先 ready，
但不会先进入模型历史，仍要等 `cargo test` 的结果按调用顺序落盘。

这里说的是**允许并发执行**，不是保证并发一定更快：两个命令可能争用 CPU、构建目录、锁文件或网络。
并发门只决定 async tool call 能否重叠，不能替命令判断内部资源冲突。

这解决了两类不同需求：

- 用户界面希望尽快显示哪个命令先跑完；
- 模型协议要求 `call_id` 和对应结果不能因为完成顺序不同而错位。

#### e. 并行是否可用的上游条件

`build_prompt` 设置 `parallel_tool_calls: true`（`core/src/session/turn.rs:1598`），
响应请求会把它发给模型。因此在常规 Responses 路径里，模型可以把多个独立调用放在同一响应。
Responses Lite 等路径会在 client 层关闭该提示（`core/src/client.rs:1001`）。

模型被允许并行，并不意味着执行端一定并行：

- 模型可以只返回一个调用；
- 多个工具中只要包含默认不并行的工具，就会触发独占写锁；
- MCP read-only hint、server opt-in、模型能力与 feature 都会影响最终声明；
- 用户中断可能让调用在等到锁之前就被取消。

#### f. 结果如何回灌给模型

正常路径使用 `ToolOutput::to_response_item`，再包装成 `ResponseItemEnvelope`。不同 payload 对应不同响应类型：

- function → `function_call_output`；
- custom/freeform → `custom_tool_call_output`；
- client tool search → `tool_search_output`。

如果 handler 返回 `FunctionCallError::RespondToModel`，`failure_response`
（`core/src/tools/parallel.rs:303`）会构造一条可回灌的失败结果。普通 function 和 custom tool
的 `success` 为 `false`；`tool_search` 是例外，它返回一个空的 `tool_search_output`。例如：

- 模型调用了当前工具表里不存在的名字；
- 参数 JSON 无法解析；
- MCP 服务返回一次可纠正的错误；
- pre-tool hook 拒绝了这次调用。

这些错误不会立刻终止进程，而是进入历史，让模型看到“为什么失败”，再决定改参数、换工具或停止。

`FunctionCallError::Fatal` 不同。它代表路由错误、handler 不可恢复、task join 失败等 host 级问题；
`handle_tool_call` 会转成 `CodexErr::Fatal`（`core/src/tools/parallel.rs:108`），不再伪装成一份
可以继续推理的普通工具结果。drain 阶段对这类错误使用 `error_or_panic`：debug 构建直接暴露，
release 构建记录错误。判断标准不是“命令退出码是否为 0”，而是“模型还能不能通过看到错误自行修正”。

所有成功/可纠正结果写入历史后，`try_run_sampling_request` 返回
`SamplingRequestResult { needs_follow_up: true, .. }`。外层 `run_turn` 重新捕获 step、重新发出模型请求；
历史里现在有原始调用和它对应的输出，模型可以继续下一步。

#### g. 取消、中断与 abort

每次工具调用拿到 turn cancellation token 的 child token。`handle_tool_call_with_source`
用 `tokio::select!` 同时等待：

- tool dispatch task 完成；
- cancellation token 取消。

取消先到时，会先检查 `terminal_outcome_reached`，避免“结果已经完成但取消也同时到达”把已完成结果丢掉；
仍未完成时才 `abort()` dispatch task，并合成 `AbortedToolOutput`
（`core/src/tools/parallel.rs:250`）。它会通知 lifecycle `ToolCallOutcome::Aborted`
（`core/src/tools/lifecycle.rs:122`），然后仍把一条“aborted by user”结果写回历史。

这使 turn 中断后恢复时有完整结构：调用 item 存在，对应 output 也明确标注被取消，而不是留下悬空 `call_id`。

两个 feature 改变中断体验：

- `InstantInterrupt`：`StepContext.preempt` 监听 turn 内的新用户输入；等待模型流时可以被输入抢占，
  已收到的调用仍按正常 drain 收尾，下一轮再处理新输入；
- `CodeModeInterrupt`：用户中断 turn 时，调用 `interrupt_active_cells()` 终止仍在运行的 Code Mode cell；
  task 先给 100ms 优雅退出窗口，再 abort 运行任务。

这些机制都围绕同一个原则：**取消结果也必须结构化地回到历史**。

#### h. 控制类工具与普通工具

`CoreToolRuntime::is_builtin_control_tool()` 会标记 `request_user_input`、plan、sleep、
部分异步消息工具等控制类调用。它本身不改变上面的调度算法，主要作用是生成专门的
control-tool analytics 状态（completed / rejected / failed）。

控制工具之所以看起来“阻塞”或“暂停”，来自 handler 本身：例如 `request_user_input` 要等客户端回复；
它默认也不声明并行，因此以 write lock 独占执行。控制工具和普通工具共享同一个 `ToolCall`、
取消、错误和结果回灌协议。

#### i. Code Mode：一个外层调用里套很多内层调用

普通 direct 模式下，模型请求几个工具，`FuturesOrdered` 就管理几个 future。
Code Mode 则把拓扑变成了两层：

```text
模型
  └─ exec（一个普通 tool call）
       └─ JavaScript cell
            ├─ tools.exec_command(...)
            ├─ tools.exec_command(...)
            └─ tools.view_image(...)
```

`CodeModeDispatchBroker` 接收 runtime 发来的 nested invocation，等对应 cell ready 后，
交给 `CoreTurnHost::submit_tool`；后者仍然调用 `ToolCallRuntime::handle_tool_call_with_source`，
只是 `ToolCallSource` 标成 `CodeMode { cell_id, runtime_tool_call_id }`。

这意味着嵌套调用复用了同一套 registry、并发门机制、hook、取消和 ToolOutput 转换，
但**不会各自生成一条 top-level model tool call**。模型看到的是外层 `exec` 的汇总输出；
每个 nested call 的身份和结果通过 `executed_tool_calls` 记录到请求 metadata/rollout trace。

Code Mode 的一个 cell 可以 yield，稍后由 `wait` 工具继续，因此 nested 调用可能跨多次顶层工具调用存活。
外层 `exec` future 结束不等于所有 Code Mode worker 都结束；cell 生命周期由 broker 的
dispatch gate 和 `wait`/`terminate` 管理。

#### j. 另一条账本和事件流

模型历史记录的是“继续推理需要看到的内容”；`ExecutedToolCalls`
（`core/src/tools/executed_tool_calls.rs:253`）记录的是另一份有界审计账本：

- 参数单条最多保存 8 KiB，超出就标记 truncated；
- 单次 output 最多附带的完整参数总量 32 KiB；
- pending nested call/cell 数量最多 256；
- recorder 只记录，不 dispatch、不 await。

工具事件也分成两条：

- `ToolEmitter` 产生用户可见的 command/patch begin/end 事件（`core/src/tools/events.rs:215`）；
- `ToolLifecycleContributor` 提供 host 观察生命周期、MCP result、timing 的 hook
  （`core/src/tools/lifecycle.rs:21`）。

它们都可能按真实完成顺序出现，和按调用顺序写入模型历史是两套不同的“顺序”。

#### k. 循环何时继续，何时结束

一次采样得到工具调用时，`needs_follow_up` 至少为 `true`。drain 完成后，
`run_turn` 回到 turn 级循环，重新采样。

没有工具调用时还有两个条件：

- `ResponseEvent::Completed.end_turn == false` 仍会触发继续（`core/src/session/turn.rs:2995`）；
- steer/queued input 非空也会让 turn 继续。

只有没有新工具调用、模型允许结束、且没有待处理输入时，turn 才进入 Stop hooks 和最终收尾。
所以“工具循环”不是独立于 turn 的 daemon，而是 `run_turn` 内部围绕采样和工具结果的一轮轮往返。

### ④ 取舍

**为什么等流结束才执行工具。** 立即执行能更早返回第一个结果，但模型可能还在继续追加工具调用；
先完整收集调用，能让 host 在执行前知道这一批操作的全貌。代价是所有工具都要等模型响应结束后才启动，
首字节和首个工具结果之间多了一段串行等待。

**为什么并行执行却按顺序回灌。** `FuturesOrdered` 同时满足吞吐和协议确定性：底层 future 并发推进，
对外仍按调用顺序产出。代价是快调用必须等前面的慢调用才能进入历史，实际已完成结果会被暂存。

**为什么用一把读写锁，而不是按文件或资源锁。** 工具只知道自己的粗粒度并行声明，host 并不知道
每条 `exec_command` 会访问哪些路径；用一把 gate 简单且可预测。代价是两个完全独立的只读工具，
只要其中一个没声明并行，也会被串行化；反过来，两个并行 shell 仍可能互相修改同一文件。
`cargo test` 和 `rg` 可以重叠运行，也只表示两个进程都被允许启动，不表示 `rg` 是由测试派生出来的后台任务。

**为什么没有固定并发 semaphore。** 模型一次响应里的工具数量本身有限，工具也各自声明并行能力。
再加一层固定上限会增加调度延迟并掩盖真正的问题。代价是极端响应可能一次启动很多 read-lock 工具，
需要靠响应上限、hook、policy 和 provider 约束共同控制。

**为什么错误要分成“回给模型”和“Fatal”。** 模型需要知道参数写错、名字写错、服务临时拒绝，
这样才能修正；host 内部路由或 task 损坏则没有可纠正的模型动作。把所有错误都回给模型会制造无效重试，
把所有错误都 Fatal 又会让一次普通命令失败直接杀掉 turn。

**为什么 Code Mode 不生成多个顶层调用。** 一个 `exec` 脚本可以把控制流、过滤和多次工具调用的组合
留在 runtime 内，减少模型往返和上下文体积。代价是外层输出变成汇总视图，nested 身份必须靠 metadata
和 trace 保留；取消和 cell 生命周期也更复杂。

回到主流程：§7 结束时，所有本批工具结果已经按模型调用顺序写入历史，turn 要么进入下一次采样，
要么正常结束。下一节转向这条链路中最敏感的部分：哪些调用在执行前会被审批、execpolicy、
sandbox 或 Guardian 拦下，以及这些决定如何变成模型可见的结果。

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
