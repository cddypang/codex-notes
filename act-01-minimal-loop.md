# 第一幕：最小闭环 —— 让模型能说话

> **幕目标**：走通从进程启动到一个 turn 结束的最小完整路径。
> 读完后你应该能回答：一个用户输入从进入 Codex 到拿到回复，中间经过了哪些结构，
> 以及为什么这里每个结构都必须存在。
>
> **前置依赖**：请先读 [第零幕：预备知识](act-00-prerequisites.md)——进程结构、核心结构、术语都在那里。
>
> **状态**：第 1–5 节正文全部完成，第一幕完结。

---

## 目录

- [1. 启动与线程模型](#1-启动与线程模型)
- [2. turn 生命周期](#2-turn-生命周期)
- [3. 模型接入](#3-模型接入)
- [4. 上下文组装](#4-上下文组装)
- [5. world state 与增量渲染](#5-world-state-与增量渲染)
- [幕末收获](#幕末收获)

---

## 1. 启动与线程模型

### ① 问题：为什么不能是一个 `while` 循环

一个最朴素的 agent 主循环长这样：

```text
while true {
    let input = read_user_input();
    let reply = call_model(history, input);
    print(reply);
}
```

这个循环在 Codex 的规模下会在四个地方崩掉。

**第一，前端不止一个。** TUI 只是其中一个入口，还有 IDE 扩展、云端任务、以及非交互的 `codex exec`。
如果循环和终端绑在一起，每个前端都要重新实现一遍上下文组装、工具执行、审批、重试。
所以循环必须和 UI 分离。

**第二，会话要能被续上。** `codex resume --last` 要求历史脱离进程存在。
所以"对话历史"必须有一个独立的持久载体，而不是内存里的一个 `Vec`。

**第三，一个进程里要同时跑多条会话。** 子智能体、side request、并行会话都是这个需求。
所以循环需要有实例身份，而不是一个全局单例。

**第四，持久实体和内存运行时必须分开。** 同一份历史可以被恢复、被 fork、被并发读取，
但同一时刻只能有一个"正在跑"的运行时。

结论：需要**两个东西**——一个持久身份，一个运行时容器。
这就是 Codex 里 `ThreadId` 和 `Session` 的分工，也是整个架构的地基。

### ② 代码入口

跟的是**交互式启动 + `--no-daemon`** 这条分支。下面这条链是**可以一步步跳的**：
每一行都是 `文件:行 -> 函数名`，按顺序在编辑器里打开就能走到底。

为什么选这条：**不带参数启动时，app-server 默认以 daemon 形式跑在另一个进程里**，
调用链在那里就断了。先跟单进程分支，链路才是连续的。（`--oss`、`--profile`、
自定义 `-c` 等也走同一条分支。）

这条链会依次穿过 [第零幕](act-00-prerequisites.md)「六类进程」表格里的第 1 类（`codex` 主进程）、
第 2 类（arg0 分派）、第 3 类（app-server），最后落到 core。

第 21 步会决定 **app-server 起在哪个进程**；daemon 分支的进程结构见
[第零幕：进程与角色](act-00-prerequisites.md#进程与角色)。

#### 第 0 步：进程入口（从 `fn main` 开始）

```text
1. cli/src/main.rs:1013                    fn main
2. cli/src/main.rs:1016                      arg0_dispatch_or_else(|arg0_paths| ...)
3. arg0/src/lib.rs:224                     arg0_dispatch_or_else
4. arg0/src/lib.rs:232                       arg0_dispatch()          # 多调用二进制分派
5. arg0/src/lib.rs:238                       std::thread::Builder::new()
6. arg0/src/lib.rs:243                         runtime.block_on(run_main_with_arg0_guard(..))
7. arg0/src/lib.rs:255                     run_main_with_arg0_guard  ->  调用闭包 main_fn(paths)
8. cli/src/main.rs:1018                    （闭包体）cli_main(arg0_paths, remote_control_disabled)
9. cli/src/main.rs:1023                    cli_main
10. cli/src/main.rs:1033                     MultitoolCli::parse()
11. cli/src/main.rs:1078                     match subcommand { None | Some(Subcommand::Agents(_)) => { ... } }
12. cli/src/main.rs:1122                       run_interactive_tui(...)
```

这里有四个不显眼但重要的点：

1. **`main` 本身几乎不做事**（`cli/src/main.rs:1013`）。它只做三件事：
   初始化 build info、读一个环境变量、然后把自己交给 `arg0_dispatch_or_else` 的闭包。
   所有真实逻辑都在 `cli_main`（`:1023`）。
2. **`arg0_dispatch`**（`arg0/src/lib.rs:232`）是"多调用二进制"分派：
   同一个可执行文件通过 `argv[0]` 的名字决定扮演什么角色
   （例如沙箱辅助进程）。这是 Codex 只需要分发一个二进制的原因。
3. **主逻辑跑在专门起的线程上**，不是主线程（`arg0/src/lib.rs:238-248`）。
   代码注释解释了原因：
   *"Run the async entry point on a thread with the same stack budget as Tokio workers;
   `Runtime::block_on` otherwise runs the top-level future on the caller's OS stack."*
   ——因为后面会有几层 `Box::pin` 专门用来避免大 future 压栈，这条约定从入口就开始了。
4. **`run_interactive_tui` 有三个调用点**（`:1122` / `:1455` / `:1542`）。
   其中 `:1122` 是 `None | Some(Subcommand::Agents(_))` 这个分支的出口，
   也就是"不带子命令"和 `codex agents` 共用的一条路；我们走的是它。

#### 第 1 步：进入 TUI

```text
13. cli/src/main.rs:2377                    run_interactive_tui
14. cli/src/main.rs:2443                       let start_tui = || { ... }
15. cli/src/main.rs:2444                         codex_tui::run_main(interactive.clone(), ...)
16. cli/src/main.rs:2451                       run_tui_with_recovery(start_tui, ...)
17. tui/src/lib.rs:1086                     run_main
18. tui/src/lib.rs:1096                       startup_orchestration::run_main_inner
19. tui/src/startup_orchestration.rs:10     run_main_inner
20. tui/src/startup_orchestration.rs:863      run_ratatui_app(...)
```

第 14 步把 `codex_tui::run_main` 包成闭包再交给 `run_tui_with_recovery`
（`cli/src/main.rs:2454`），后者是个重试循环——`local_state_db` 被锁或损坏时会尝试自动备份重建。
这也解释了为什么要用闭包而不是直接调用：**TUI 可能要启动两次**。

`run_ratatui_app`（`tui/src/lib.rs:1127`）里，21 和 22 之间还夹着约 800 行的启动期决策
（`tui/src/lib.rs:1214–2030`）：

```text
21. tui/src/lib.rs:1197    start_app_server(...)  ->  AppServerClient
      tui/src/lib.rs:551    start_app_server          # 按 target 决定 server 起在哪
      │
      │  target == Embedded（本文档跟的分支）
      ├─  tui/src/lib.rs:284      start_embedded_app_server
      │     tui/src/lib.rs:673        start_embedded_app_server_with
      │       app-server-client/src/lib.rs:355   InProcessAppServerClient::start
      │         app-server/src/in_process.rs:377   start   # 服务端在这里诞生（同进程）
      │     -> AppServerClient::InProcess
      │
      │  target == LocalDaemon / Remote
      └─  app_server_connection::connect(target)           # 连到对方进程里的 server
          -> AppServerClient（远程形态）

      ↓ 启动期的一次性决策（tui/src/lib.rs:1214–2030）
        ① 包装成 AppServerSession（任一步失败都恢复终端并返回）
        ② 查登录状态 → 决定要不要跑 onboarding
        ③ 解析 SessionSelection：新会话 / resume / fork / agents overview
        ④ 选择落定后再重载一次配置（注释：the last possible reload）
        ⑤ 目录信任检查（remote 与本地走不同路径）
        ⑥ 收尾 UI 状态：主题、alt-screen、首次运行标记
        ⑦ 并行预取 bootstrap 与 hooks 条目
        ⑧ hooks 审查

22. tui/src/lib.rs:2034    App::run(...)             # 两种结果都走到这里
```

这八件事必须在进循环之前定下来，因为进去之后再改代价很高：改配置要重建 chat widget、
换主题要重绘、切会话要重启会话运行时。而且它们几乎都包在
`startup_draft.run_until(&mut tui, ...)` 里——每处耗时操作都在渲染循环的照看下执行，
**所以启动时看到的是一个能响应的画面，不是黑屏等待**。

注意这**不是调用链分叉**：`start_app_server` 无论走哪条子路径，返回的都是同一个
`AppServerClient`；`run_ratatui_app` 拿到后统一包成 `AppServerSession`，再进 `App::run`。
变的只是"server 跑在哪个进程"，不是"接下来调用谁"。

而且连"走哪条子路径"都不是静态的：daemon 连不上、且 `allow_embedded_fallback` 为真时，
函数内部会**把 `target` 改回 `Embedded`**，然后落到末尾起同进程 server——这就是第零幕里
说的"可选附着会降级"。

注意 `start_embedded_app_server` 本身只是把调用**装箱下沉**一层
（`Box::pin(start_embedded_app_server_with(..., InProcessAppServerClient::start))`），
真正的客户端启动在 `InProcessAppServerClient::start`（`app-server-client/src/lib.rs:355`），
它在同进程里拉起 `app-server/src/in_process.rs:377` 的 `start`。
所以"TUI 调 core"这件事，物理上就是函数调用，没有 IPC。

#### 第 2 步：TUI 发起 `thread/start`

接着上一步（22）留下的 `App::run`（`tui/src/app/startup.rs:178`）继续：

```text
23. tui/src/app/startup.rs:424              spawn_startup_thread_start(...)   # 调用点
24. tui/src/app/startup.rs:16             spawn_startup_thread_start        # 定义
      -> tokio::spawn(async move { ... })  # 注意：是异步起的，不阻塞 UI 渲染
25. tui/src/app_server_session.rs:1682    start_thread_with_request_handle
26. tui/src/app_server_session.rs:233     request_thread_start_with_history_fallback
27. tui/src/app_server_session.rs:241     request_handle.request_typed(ClientRequest::ThreadStart { .. })
```

几个细节：

- 第 24 步用 `tokio::spawn`，因为要先把终端让出来渲染启动画面；
  结果通过 `AppEvent::StartupThreadStarted`（`tui/src/app_event.rs:534`）回到事件循环，
  由 `tui/src/app/event_dispatch.rs:268` 处理。
- 第 25 步之前，参数由 `thread_start_params_from_config`（`tui/src/app_server_session.rs:2035`）
  从 `Config` 生成——这是客户端配置变成协议参数的地方。
- `request_thread_start_with_history_fallback` 里有个循环：如果服务端不支持分页历史，
  会清掉 `history_mode` 重试一次（`:255-261`）。

#### 第 3 步：服务端侧分派

```text
28. app-server/src/message_processor.rs:1284        ClientRequest::ThreadStart { params, .. } => ...
29. app-server/src/request_processors/thread_processor.rs:526   ThreadProcessor::thread_start
30. app-server/src/request_processors/thread_processor.rs:1127  thread_start_inner
31. app-server/src/request_processors/thread_processor.rs:1315  thread_start_task
32. app-server/src/request_processors/thread_processor.rs:1498  .start_thread(StartThreadOptions { .. })
```

第 32 步之前的几行（`:1486-1494`）值得看一下：先 `reserve_thread_id()`，
再把创建者连接注册到这个 thread 上，注释解释了原因——
*"Startup can prewarm the Responses socket before start_thread returns."*

第 32 步同时是 `ThreadStartSource` → `InitialHistory` 的映射点（`:1500-1505`），
也就是 `/clear` 与普通启动的分岔处：

```rust
initial_history: match session_start_source
    .unwrap_or(codex_app_server_protocol::ThreadStartSource::Startup)
{
    ThreadStartSource::Startup => InitialHistory::New,
    ThreadStartSource::Clear   => InitialHistory::Cleared,
},
```

#### 第 4 步：core 侧建 thread

```text
33. core/src/thread_manager.rs:1036   ThreadManager::start_thread
34. core/src/thread_manager.rs:1109   start_thread_inner
35. core/src/thread_manager.rs:1154     self.state.spawn_thread(request)
36. core/src/thread_manager.rs:2040   ThreadManagerState::spawn_thread   # impl 块从 :1606 开始
37. core/src/thread_manager.rs:2270     Session::spawn(SessionSpawnArgs { .. })
38. core/src/session/mod.rs:512       Session::spawn
39. core/src/session/mod.rs:540       Session::spawn_internal            # 真正干活的地方
```

注意第 33 → 36 步跨了两个类型：`ThreadManager`（`impl` 从 `:519` 开始）只是个外壳，
真正的注册表在 `ThreadManagerState`（`impl` 从 `:1606` 开始）。第 35 步是这一跳的证据。

`Session::spawn_internal`（第 39 步）就是"把一条会话从零建起来"，依次做六件事：

```text
1. 建两个通道：提交通道（bounded 512）+ 事件通道（unbounded）
2. 解析启动配置：模型 fallback、token budget、base instructions（三级优先级）
3. 决定历史模式、动态工具、插件选择
4. 组装 SessionState（可变状态）与 SessionServices（依赖容器）
5. 起后台 submission_loop —— 这条会话的心跳
6. 返回 (Arc<Session>, SessionIo)
```

到这里第一次出现几个后面会反复用的词，先给最小定义：

| 名字 | 是什么 |
|---|---|
| `Session` | 会话的**内存运行时**（③b 展开） |
| `SessionIo` | 与外界的**两条通道**：`Op` 进去、`Event` 出来（③c 展开） |
| `Op` | 外部提交给会话的操作 |
| `Event` | 会话向外发出的事件 |
| `submission_loop` | 后台循环，消费 `Op` 并分派（第 2 节展开） |

**返回链是往回走的**，第 39 步的返回值一路冒泡回 TUI：

```text
39 返回 (Arc<Session>, SessionIo)
  → 36  组装成 NewThread（core/src/thread_manager.rs:172）
    → 34  原样返回
      → 15  app-server 的 thread_start_task 拿到 NewThread，构造 ThreadStartResponse
        → 27  TUI 的 request_thread_start_with_history_fallback 收到响应
```

中间只在第 36 步换过一次类型：

```rust
pub struct NewThread {
    pub thread_id: ThreadId,
    pub thread: Arc<CodexThread>,        // ← 注意不是 Arc<Session>
    pub session_configured: SessionConfiguredEvent,
}
```

**外部拿到的是 `Arc<CodexThread>`，不是 `Arc<Session>`**——对外契约和内部运行时是分开的，
见 ③a。`thread_id` 被单独拎出来，是因为它才是持久句柄。

#### 两个容易踩的坑

- `tui/src/lib.rs` 里有**两个** `start_app_server` 调用点（`:624` 和 `:651`），
  那是给 picker（`codex resume` 的选择界面）用的，不是交互式启动路径。交互式走的是 `:1197`。
- `session/mod.rs` 里的 `INITIAL_SUBMIT_ID`、`Session::spawn`、`spawn_internal` 都在同一个文件，
  行号跨度很大（`:503` / `:512` / `:540`），不要按文件名跳，要按行号跳。

### ③ 关键设计

#### a. 四个层次，四种生命周期

| | `ThreadId` / `CodexThread` | `Session` |
|---|---|---|
| 可见性 | `pub`（对外契约） | `pub(crate)`（内部） |
| 生命周期 | 跨进程，由 rollout 文件承载 | 进程内，随 runtime 存活 |
| 职责 | 持久句柄、对外 API | 依赖容器、状态机宿主 |

`thread-store` 的模块注释把第一列的定义写得很死：

> Application code should treat `ThreadId` as the only durable thread handle.

而 `CodexThread` 只是 `Session` 的一层薄封装：

```rust
pub struct CodexThread {
    pub(crate) session: Arc<Session>,
    pub(crate) io: SessionIo,
    pub(crate) session_source: SessionSource,
    startup_metadata: ThreadStartupMetadata,
    rollout_path: Option<PathBuf>,        // 指向持久文件
    ...
}
```

把后面几节会出现的东西一起放进来，一共四个层次，**粒度依次递减，换下一个不会换上一个**：

```text
ThreadId        持久：磁盘上的 rollout 文件，跨进程存在
   └─ Session   每次加载一个：一个进程里的一次实例化
        ├─ services / state     会话级，跨多个 turn 存活
        └─ active_turn          每轮换
             └─ task            每轮换（与 turn 基本等价，第 2 节展开）
```

两个结论各自有代码依据。

**① 换 task 不换 `Session`。** `spawn_task` 只动 `active_turn` 字段：

```rust
self.abort_all_tasks(TurnAbortReason::Replaced).await;   // 清掉旧 task
self.clear_connector_selection().await;
self.start_task(turn_context, input, task).await;        // 装上新 task
```

而 `abort_all_tasks` 最后调的是 `take_active_turn`，里面只有一句 `active.take()`——
置空的是 `active_turn`，`Session` 对象本身没动。

反过来说更清楚：如果 `Session` 跟着 task 换，那 `SessionServices` 里的东西（MCP 长连接、
rollout writer、扩展的会话级状态、模型的 WebSocket）每轮都要重建。**`Session` 存在的意义
就是承载这些跨 turn 复用的东西。**

**② thread 和 `Session` 的生命周期并不一致。** thread 是持久实体，`Session` 只是它的
一次实例化：

| 事件 | thread | `Session` |
|---|---|---|
| 新建会话 | 新建 | 新建 |
| 换 turn / 压缩 | 不变 | **不变** |
| 进程退出（embedded） | 停止运行（文件还在） | **析构** |
| `codex resume` | **同一条**（同一个 `ThreadId`） | **新建** |
| `codex fork` | **新建** | 新建 |

`resume` 那一行是关键：**同一个 `ThreadId`，新的 `Session`**。这正是
`ThreadLifecycleContributor::on_thread_resume` 存在的原因——扩展要在新 `Session` 上
重新水合自己的状态（第三幕 §9 会讲）。

所以准确的说法是：**`Session` 是 thread 在某一个进程里的一次实例化。一个 thread 一生可以有
多个 `Session`，但任一时刻最多一个**——由第零幕那条"同一个 `CODEX_HOME` 只能有一个 owner"
的约束保证。

#### b. `Session` 的边界由它自己的注释定义

```rust
/// Context for an initialized model agent
///
/// A session has at most 1 running task at a time, and can be interrupted by user input.
pub(crate) struct Session {
    pub(crate) thread_id: ThreadId,
    ...
    pub(crate) active_turn: Mutex<Option<ActiveTurn>>,
    pub(crate) input_queue: InputQueue,
    pub(crate) services: SessionServices,
}
```

两句话给出两条硬约束：

- **同时最多一个 task** → 决定了 turn 的并发模型（第 2 节）
- **可被用户输入打断** → 决定了 `input_queue` 和 steer 机制的存在

**"at most 1" 是上限，不是恒等式。** 展开字段看是两层 `Option`：

```rust
pub(crate) struct Session {
    ...
    pub(crate) active_turn: Mutex<Option<ActiveTurn>>,   // 第一层：可能整个会话空闲
}

pub(crate) struct ActiveTurn {
    pub(crate) task: Option<RunningTask>,                // 第二层：有 turn 但没在跑
    pub(crate) turn_state: Arc<Mutex<TurnState>>,
}
```

于是有三种状态：

| 状态 | `active_turn` | `.task` |
|---|---|---|
| 完全空闲 | `None` | — |
| 有 turn，正在跑 | `Some` | `Some` |
| 有 turn，正在收尾 | `Some` | `None` |

**这条不变量是靠机制维持的，不是靠调用方自觉**：`spawn_task` 的第一件事就是
`abort_all_tasks(TurnAbortReason::Replaced)`（第 2 节展开）。

另有一条约束藏在字段注释里：

> The set of enabled features should be invariant for the lifetime of the session.

特性开关在 session 生命周期内不可变。这解释了一个行为：为什么 `/clear` 是**开一个新 thread**
而不是就地重置当前会话——因为它需要换一套 feature 配置。

#### c. IO 是双向的：`Op` 进，`Event` 出

```rust
pub(crate) struct SessionIo {
    pub(crate) tx_sub: Sender<Submission>,
    pub(crate) rx_event: Receiver<Event>,
    pub(crate) agent_status: watch::Receiver<AgentStatus>,
    pub(crate) session_loop_termination: SessionLoopTermination,
}
```

四个字段各有讲究：

- `tx_sub` 是 `Sender<Submission>`，不是 `Sender<Op>`——提交带一个信封（含 id、trace 等），
  这样事件才能归属回具体的提交。
- `rx_event` 是事件流出口，TUI 就靠它渲染。
- `agent_status` 是个 `watch` 通道，让外部随时读当前状态而不阻塞。
- `session_loop_termination` 是一个共享 future，让**多个**调用者等待同一个关闭过程——
  `shutdown_and_wait()` 正是靠它实现"提交 Shutdown 后等循环真正退出"。

后台真正跑的是这一行（`core/src/session/mod.rs:936`）：

```rust
submission_loop(session_for_loop, configured_config, rx_sub)
```

这是整个 agent 的心跳：所有 `Op` 都在这里被消费、分派到对应的 task。

另一个细节：`INITIAL_SUBMIT_ID = ""`（`core/src/session/mod.rs:503`）——
空字符串表示"不是由某次提交触发的事件"，比如启动期的警告。

#### d. `SessionServices`：一次性注入的依赖容器

```rust
pub(crate) struct SessionServices {
    pub(crate) mcp_runtime: Arc<McpRuntime>,
    pub(crate) unified_exec_manager: UnifiedExecProcessManager,
    pub(crate) exec_policy: Arc<ExecPolicyManager>,
    pub(crate) auth_manager: Arc<AuthManager>,
    pub(crate) models_manager: SharedModelsManager,
    pub(crate) model_client: ModelClient,
    pub(crate) extensions: Arc<ExtensionRegistry<Config>>,
    pub(crate) session_extension_data: ExtensionData,
    pub(crate) thread_extension_data: ExtensionData,
    pub(crate) skills_service: Arc<HostSkillsService>,
    pub(crate) plugins_manager: Arc<PluginsManager>,
    pub(crate) agent_control: Arc<dyn AgentControl>,
    pub(crate) thread_store: Arc<dyn ThreadStore>,
    pub(crate) session_telemetry: SessionTelemetry,
    ...
}
```

关键点：**这些依赖在 session 创建时统一注入，turn 只读**。

这也解释了一个后面会反复出现的模式：扩展的私有状态放 `ExtensionData`
（`session_extension_data` / `thread_extension_data`），而不是往 core 里加字段。
第 10 节会展开。

#### e. 启动时的一次性决策

`Session::spawn_internal` 里有一批"只做一次"的解析，其中最有代表性的是 base instructions 的三级优先级：

```rust
let base_instructions = config
    .base_instructions
    .clone()                                        // 1. 显式配置覆盖
    .or_else(|| conversation_history.get_base_instructions().map(|s| s.text))  // 2. 历史里记录的
    .unwrap_or_else(|| render_model_instructions(&model_info));                // 3. 按当前模型渲染模板
```

"配置 vs 历史"的冲突在这里第一次出现，后面在权限、插件选择、token budget 上都会重复同一个问题：
**恢复一条旧会话时，新配置该覆盖多少旧状态？** Codex 的答案是逐项决定，而不是一刀切。

### ④ 取舍

**为什么 thread 不能是无状态的纯函数？**

因为 session 需要长期持有这些东西：MCP 长连接、文件 watcher、后台任务（记忆抽取、code mode host）、
审批状态、token 用量累计。它们既不属于某一次 turn，也不适合每轮重建。

代价是生命周期管理变复杂：`shutdown_and_wait()` 要保证循环真的退出，
`SessionStartup::cleanup()` 还要处理"启动了但没交接成功"的中间态。这种边界情况在
`core/src/session/startup.rs` 里有专门处理。

**为什么 `Session` 是 `pub(crate)`，而 `CodexThread` 是 `pub`？**

外部（app-server、TUI、集成测试）只能通过 `CodexThread` 交互。
好处是 `Session` 的内部结构可以自由重构而不破坏 API；
代价是每加一个外部能力，都得在 `CodexThread` 上开一个口子。

**为什么中间要隔一个 app-server？**

表面上多了一层。但它的收益是：TUI、IDE、云端、`codex exec` 共享同一条实现路径。
而且 TUI 用的是**进程内** app-server，实际没有 IPC 开销。
代价是协议变成了兼容性承诺——AGENTS.md 里专门有一节讲 breaking changes。

**总的代价：命名债。**

用户界面上的 "session"、代码里的 thread、以及代码里另一个叫 `Session` 的运行时对象，
三者指的是不同东西。这个歧义是历史遗留（thread 曾经叫 conversation），
`codex-agent-architecture.md` 第 11 节专门做了对照表。

### 建议的阅读顺序

按这个顺序过一遍代码最省事：

0. `cli/src/main.rs:143` 的 `Subcommand` 枚举 —— 先看这个程序能扮演哪些角色
1. `core/src/session/session.rs` —— 只看 `Session` 结构体及其文档注释（建立边界感）
2. `core/src/codex_thread.rs` —— 看 `CodexThread` 持有什么，理解"薄封装"
3. `core/src/thread_manager.rs:1036-1130` —— 启动入口
4. `core/src/session/mod.rs:512-700` —— `spawn_internal` 的前半段（一次性决策）
5. `core/src/state/service.rs` —— `SessionServices` 全貌

---

## 2. turn 生命周期

本节分三部分：

1. **turn 是什么** —— 定义、在全局中的位置、谁在维护它、生命周期、上下游交互
2. **turn 怎么从 TUI 产生** —— 以及它怎么和 app-server、core 里的 turn 对应上
3. **core 怎么处理它** —— 三层循环、结束判定、超限压缩、形态与中断

---

### ① turn 是什么

#### a. 定义

"turn" 在第 1 节里露过面，但**从未定义**——那里的 `Session` 注释提到 "at most 1 running task"，
调用链也只走到 `thread/start`，没有进入 turn 内部。

> **一次 turn = 一次用户提交所触发的全部工作。**
> 从输入被接受开始，到模型不再要求继续（既没有新的工具调用，也没有待处理输入）为止。
> **一次 turn 内部可以包含多次模型请求。**

"可以包含多次模型请求"这一条是定义里最容易被忽略的部分：模型要求工具调用时，执行完要把
结果发回去再问一轮，这些都发生在同一个 turn 内。

#### b. 它在全局的位置

第 1 节 ③a 给过四层嵌套，turn 是最里面那一层：

```text
ThreadId        持久：磁盘上的 rollout 文件，跨进程存在
   └─ Session   每次加载一个：一个进程里的一次实例化
        ├─ services / state     会话级，跨多个 turn 存活
        └─ active_turn          每轮换
             └─ task            每轮换
```

`task` 和 `turn` **不是同义词，但基本一一对应**：task 是 turn 的运行载体，一个 turn 由一个
task 执行。两者只在两处有偏差：

- **一个 task 内部可能跑多次 `run_turn`**（同一个 `turn_id`），见 ③a；
- **`TaskKind::Compact` / `Review` 从 turn 视角看也是 turn**，只是换了 task 实现，见 ③d。

| | turn | task |
|---|---|---|
| 是什么 | **逻辑单位**：一次提交 → 最终答复 | **运行时单位**：跑在 tokio 后台任务上的 future |
| 标识 | `turn_id`（即 `TurnContext.sub_id`） | `Arc<dyn AnySessionTask>` + `AbortOnDropHandle` |
| 分类 | — | `TaskKind::{Regular, Review, Compact}`（`state/turn.rs:68`） |

#### c. 谁在维护它

四个结构，按存活范围从大到小：

| 结构 | 位置 | 存活范围 | 装什么 |
|---|---|---|---|
| `TurnContext` | `session/turn_context.rs:305` | 整个 turn | turn 级配置、`terminal_error`、插件选择、extension data |
| `TurnState` | `state/turn.rs:89` | 整个 turn | **可变**状态：待审批、待提问、tool 调用计数、token 用量 |
| `StepContext` | `session/step_context.rs` | **单次采样请求** | 模型、工具表、MCP 绑定、AGENTS.md、环境快照 |
| `RunningTask` | `state/turn.rs:74` | 整个 turn | 取消令牌、事件句柄、`Arc<TurnContext>`、若干 RAII 守卫 |

前两个是"turn 知道什么"，第四个是"turn 在运行时握着什么"。三者之间有一句关键约束，
写在 `TurnContext` 里：

> Call state at turn creation; model requests use the `StepContext` snapshot.

也就是：**turn 级的设置可以被中途改，但已经发出去的请求用当时快照的那一份**。

`StepContext` 的模块注释把它的定位说得更直接：

> Request-scoped settings and capabilities, including the durable context snapshot.

这四者挂在哪？`Session` 上的一个字段：

```rust
pub(crate) struct Session {
    ...
    pub(crate) active_turn: Mutex<Option<ActiveTurn>>,   // 第一层：可能整个会话空闲
}

pub(crate) struct ActiveTurn {
    pub(crate) task: Option<RunningTask>,                // 第二层：有 turn 但没在跑
    pub(crate) turn_state: Arc<Mutex<TurnState>>,
}
```

`RunningTask` 的字段值得看一眼，它展示了"一个 turn 运行时握着什么"：

```rust
pub(crate) struct RunningTask {
    pub(crate) done: Arc<Notify>,
    pub(crate) kind: TaskKind,
    pub(crate) task: Arc<dyn AnySessionTask>,
    pub(crate) cancellation_token: CancellationToken,
    pub(crate) handle: AbortOnDropHandle<()>,
    pub(crate) turn_context: Arc<TurnContext>,
    pub(crate) _agent_execution_guard: Option<AgentExecutionGuard>,
    pub(crate) _diagnostics_guard: GaugeGuard,
    pub(crate) _timer: Option<codex_otel::Timer>,
}
```

最后那个 `_timer` 的注释写着"记录在 task drop 时的完整时长"——又一次 RAII 计时。

**"at most 1 running task" 是上限，不是恒等式**（第 1 节 ③b 已展开）：两层 `Option` 对应
三种状态——完全空闲 / 有 turn 且正在跑 / 有 turn 正在收尾。

#### d. 生命周期：五个阶段

```text
产生   start_or_steer → spawn_task                     （② 展开）
运转   RegularTask::run → run_turn 循环                （③a 展开）
收敛   needs_follow_up 为假 → 跑 Stop hook             （③b 展开）
收尾   on_task_finished（tasks/mod.rs:630）→ 发事件 + 落盘
销毁   RunningTask drop → turn_store 丢弃，turn 结束
```

「收尾」这一步的 `on_task_finished` 负责三件事：把 `TurnCompleteEvent` 或 `TurnAbortedEvent`
发给客户端、落盘最后的历史、清掉 `active_turn`。「销毁」不需要显式代码——`turn_store` 是跟着
turn 走的 `ExtensionData`，`RunningTask` 一 drop 它就没人引用了。

#### e. 上下游交互

把 turn 放在数据流中间看，它同时被上游驱动、又驱动下游：

```text
上游（谁给 turn 送东西）
  ├─ 用户输入        turn/start → Op::TurnInput
  ├─ 中途插话        turn/steer → 折进当前 turn（steer）
  ├─ 子 agent 消息   Op::InterAgentCommunication
  ├─ hook 注入        UserPromptSubmit / PostToolUse 的 additionalContext
  └─ 工具结果回灌      FunctionCallOutput

turn（本文主角）

下游（turn 驱动谁）
  ├─ 模型请求        run_sampling_request → ModelClient（第 3 节）
  ├─ 工具执行        ToolCallRuntime（第二幕）
  ├─ 持久化          rollout / world state / TurnContextItem
  └─ 事件流          Event → app-server → TUI
```

这张图里有两个方向都存在的通道：**`Op` 进、`Event` 出**（第 1 节 ③c）。

---

### ② turn 怎么产生：从 TUI 到 core

TUI 里有六个会产生 turn 的入口，但它们**全部汇聚到同一个出口**：

```text
[场景 1] 输入框回车          composer → submit_user_message
[场景 2] 排队消息被派发       maybe_send_next_queued_input → slash_dispatch 的 drain 分支
[场景 3] 斜杠命令自带提示词    /init 等 → submit_user_message(INIT_PROMPT)
[场景 4] 启动时的初始提示      codex "..." → ChatWidgetInit.initial_user_message
[场景 5] 失败恢复 / 重试       input_restore.rs:180
[场景 6] 带协作模式的提交      AppEvent::SubmitUserMessageWithMode
        │
        │  六个入口构造的是同一个东西：Op::UserTurn { items, cwd, model, ... }
        ▼
chatwidget::submit_op(op)                                    chatwidget.rs:1809
        │  按 codex_op_target 分流（两个目标）
        ├─ CodexOpTarget::Direct(tx)  → tx.send(op)             # 旧路径
        └─ CodexOpTarget::AppEvent    → AppEvent::CodexOp(op)   # 现路径
              │
              ▼
try_submit_active_thread_op_via_app_server(app_server, thread_id, &op)
    tui/src/app/thread_routing.rs:659
        │  match op { Interrupt => …, UserTurn { .. } => … }
        ▼
AppCommand::UserTurn { .. } 分支                              thread_routing.rs:736
        │
        ▼
app_server.turn_start(thread_id, client_user_message_id, items, cwd, …)
    tui/src/app_server_session.rs:1288                        thread_routing.rs:874
```

三个关键点：

1. **`app_server.turn_start` 在整个 TUI 里只有一个调用点**（`thread_routing.rs:874`）。
   六个入口、一个出口。
2. **六个入口构造的是同一个东西**：`Op::UserTurn`。区别只在 `items`、`cwd`、
   `collaboration_mode` 这些字段怎么填。
3. **`submit_op` 有两个目标**（`chatwidget.rs:1836`）：`Direct` 是旧路径直接写通道，
   `AppEvent` 是现在的路径——交给 app 层统一处理。所以下面的链从 `AppEvent` 开始。

#### a. 六个入口分别是什么

| # | 触发方式 | 入口 | 说明 |
|---|---|---|---|
| 1 | 输入框回车 | `chatwidget/input_submission.rs` 的 `submit_user_message*` | 主路径 |
| 2 | 排队消息被派发 | `chatwidget/slash_dispatch.rs:1095` 一带 | 上一轮在跑时提交的消息会排队，空闲后逐条派发 |
| 3 | 斜杠命令自带提示词 | `slash_dispatch.rs:292`（`/init`）等 | 命令自己构造提示词再走提交 |
| 4 | 启动时的初始提示 | `codex "帮我修一下这个 bug"` | 参数进 `ChatWidgetInit.initial_user_message`，**先排队**，会话配置好后走场景 2 的派发 |
| 5 | 失败后恢复 | `chatwidget/input_restore.rs:180` | 把用户消息恢复并重新提交 |
| 6 | 指定协作模式的提交 | `app/event_dispatch.rs:2995` 的 `AppEvent::SubmitUserMessageWithMode` | 例如从 Plan 模式确认后提交 |

场景 1 是主路径，其余五个都是"绕过键盘但走同一条路"。`AppCommand` 里还有 `Review` /
`Compact` 等分支，它们也在同一个 match 里分流，但语义不是"用户提交一句话"，所以没列进来。

#### b. 从 `turn_start` 往下：跨进程到 core

第 1 节的链在第 39 步（`Session::spawn_internal`）把 thread 建起来，返回值一路冒泡回 TUI。
本节从那里接着走——**一句话从 TUI 出发之后发生了什么**：

```text
40. tui/src/app_server_session.rs:1288        AppServerSession::turn_start
41. tui/src/app_server_session.rs:1310          request_typed(ClientRequest::TurnStart { .. })
      ═══════ 跨进程边界（embedded 时同进程；daemon 时是 unix socket）═══════
42. app-server/src/message_processor.rs:1629   ClientRequest::TurnStart { .. } => ...
43. app-server/src/request_processors/turn_processor.rs:174   TurnProcessor::turn_start
44. app-server/src/request_processors/turn_processor.rs:522   turn_start_inner
45. app-server/src/request_processors/turn_processor.rs:652     thread.start_or_steer_turn(...)
46. core/src/codex_thread.rs:378                CodexThread::start_or_steer_turn
47. core/src/codex_thread.rs:533                  submit_turn_input_with_mode
48. core/src/session/mod.rs:1001                   SessionIo::submit_turn_input    # 生成 turn_id、装 Submission
49. core/src/session/mod.rs:990                      tx_sub.send(sub).await       # ← 写进队列
      ═══════ 进提交队列：第 1 节 ③c 讲过的 SessionIo.tx_sub，容量 512 ═══════
50. core/src/session/handlers.rs:427            submission_loop：rx_sub.recv()   # ← 从同一个队列取出
51. core/src/session/turn_input.rs:211           turn_input::handle
52. core/src/session/turn_input.rs:299             start_or_steer
53. core/src/tasks/mod.rs:272                        spawn_task         # ← turn 在这里诞生
54. core/src/tasks/mod.rs:287                          start_task
55. core/src/tasks/regular.rs:40                         RegularTask::run # ← 进入 turn 内部
56. core/src/session/turn.rs:163                           run_turn
57. core/src/session/turn.rs:1618                            run_sampling_request   # ← 一次采样请求（内部见第 3 节）
```

从第 57 步返回之后，`run_turn` 还要走完 turn 自己的收敛路径（采样内部结束后才会走到这里）：

```text
core/src/session/turn.rs:536      run_turn：match sampling_request_result
core/src/session/turn.rs:566      needs_follow_up = model_needs_follow_up || has_pending_input
core/src/session/turn.rs:757      continue             # 需要续跑（模型要工具调用 / 有 steer 输入）→ 回到 loop 顶部再采样
core/src/session/turn.rs:655      run_turn_stop_hooks  # 不需要续跑 → 先过 Stop hook（可阻断 / 可延长）
core/src/session/turn.rs:726      run_auto_compact     # 达到阈值时做 post-turn 压缩
core/src/session/turn.rs:755      break                # turn 收敛
core/src/tasks/mod.rs:630        on_task_finished     # 发 TurnComplete / TurnAborted + 落盘最后历史
core/src/session/turn.rs:839      run_turn 返回 Ok(last_agent_message)
```

五处值得注意的衔接：

- **第 41 → 42 步是进程边界**。embedded 下它只是函数调用，daemon 下是 unix socket 上的
  JSON-RPC——但两边看到的方法名完全相同，"服务端在哪"不影响这条链的形状（第 1 节 ③a）。
- **第 48 → 49 步是"装信封 → 入队"**：`submit_turn_input`（`mod.rs:1001`）只做三件事——
  生成 `turn_id`、建 oneshot、把 `Op::TurnInput` 包成 `Submission`；真正的 `send` 在下一层
  `submit_with_id`（`mod.rs:986`）里。顺带一提，`send` 前它会补上 W3C trace：

  ```rust
  pub(crate) async fn submit_with_id(&self, mut sub: Submission) -> CodexResult<()> {
      if sub.trace.is_none() {
          sub.trace = current_span_w3c_trace_context();
      }
      self.tx_sub.send(sub).await.map_err(|_| CodexErr::InternalAgentDied)?;
      Ok(())
  }
  ```

  **`tx_sub` 就是第 1 节 ③c 里那个 `SessionIo.tx_sub: Sender<Submission>`，队列在
  `mod.rs:586` 用 `async_channel::bounded(SUBMISSION_CHANNEL_CAPACITY)` 建出来，容量 512。**
  所以第 49 步和第 50 步是**同一个队列的写端和读端**——第 49 步 `send`，第 50 步 `recv`。
  因为是 bounded 队列，队列满时 `send().await` 会等，这就是背压。
- **第 48 → 50 步之间是异步的**：`submit_turn_input` 入队后 `await` 一个 oneshot 回执
  （`core/src/session/mod.rs:1022` 的 `reply_rx.await`），`submission_loop` 处理完再把结果
  写回这个 oneshot。**所以第 43 步之后的 `turn_start` 是"发出去等回话"，不是"直接调用"。**
- **第 52 → 53 → 55 步是"turn 诞生"的三拍**，值得展开：

  **第一拍（52 → 53，`spawn_task`）** —— 只做两件准备工作，然后转发：

  ```rust
  pub async fn spawn_task<T: SessionTask>(self: &Arc<Self>, turn_context, input, task) {
      self.abort_all_tasks(TurnAbortReason::Replaced).await;   // ① 先中止上一个 turn
      self.clear_connector_selection().await;                  // ② 清掉上一轮的 connector 选择
      self.start_task(turn_context, input, task).await;        // ③ 转发
  }
  ```

  第 ① 行就是"一个 thread 同时只有一个 turn"那条不变量的实现位置（③d 会再提一次）。

  **第二拍（53 → 54，`start_task` 前半段）** —— `ActiveTurn` 在这里被创建，但 `task` 还是空：

  ```rust
  let mut active = self.active_turn.lock().await;
  let turn = active.get_or_insert_with(ActiveTurn::default);   // ← active_turn 诞生
  debug_assert!(turn.task.is_none());
  ```

  接着它会建 `CancellationToken`、记下 `token_usage_at_turn_start`、把排队的 mailbox 消息
  塞进 `turn_state`，并跑一次 `TurnStartPhase::BeforeTaskRegistration` 的扩展 lifecycle。

  **第三拍（54 → 55，`start_task` 收尾）** —— 关键在这里：`task.run(...)` **不是被直接调用**，
  而是包进一个 future 交给 `tokio::spawn`：

  ```rust
  let task_future = async move {
      let task_result = task_for_run.run(Arc::clone(&session), ctx, task_input, token).await;
      ...
      sess.on_task_finished(Arc::clone(&ctx_for_finish), task_result).await;
      done_clone.notify_waiters();
  };
  let handle = tokio::spawn(storage_originator.scope(task_future));   // ← 另起一个 tokio task
  ...
  turn.task = Some(RunningTask { done, handle, kind, task, cancellation_token, ... });  // ← turn 诞生
  ```

  **"turn 诞生"的准确时刻就是最后那一行 `turn.task = Some(...)`**：`active_turn.task` 从
  `None` 变成 `Some`，而与此同时后台的 `RegularTask::run` 已经被 spawn 出去了。

  这同时解释了 54 → 55 为什么看起来是"跳"的——**它跨到了一个全新的 tokio task，
  不是同一条调用栈**。`spawn_task(...).await` 返回时，`RegularTask::run` 可能刚开始跑、
  也可能还没被调度。第 55 步往下的整条链都发生在那条新的 tokio task 上。

  （还有一处呼应：`start_task` 里发的是 `TurnStartPhase::BeforeTaskRegistration`，
  而 `RegularTask::run` 内部还会再发一次 `TurnStartPhase::RegularTaskStart`——
  这就是第零幕 `TurnStartPhase` 两个取值的实际用途。）

- **第 57 步是第 3 节的入口，不是这条链的终点**：`run_sampling_request` 内部就是"发模型请求"，
  它展开在第 3 节；等它返回，上面那段收敛路径才走完 turn——`run_turn` 按 `needs_follow_up`
  决定"再发一轮采样"还是"过 Stop hook / post-turn 压缩后收敛"，最后由 `on_task_finished`
  发事件并落盘。

#### c. 三层身份怎么对应

这条链上出现过好几个 ID，容易以为是不同的东西。实际上它们**是同一个值**：

```text
core/src/session/mod.rs:1006     let id = new_submission_id();     ← 唯一一次生成
                                        │
                                        ├─ Submission.id
                                        ├─ TurnInputSubmission::Started { turn_id }   （返回给调用方）
                                        ├─ TurnContext.sub_id                          （turn 的运行身份）
                                        └─ Event.id                                    （所有事件的归属）
```

四个名字指的是一回事：**`turn_id == submission_id == TurnContext.sub_id == Event.id`**。

`new_submission_id()` 用的是 UUIDv7，实现里留了一句警告：

> Some use cases take advantage of the fact that these are UUID7 which encodes a timestamp,
> so think carefully before changing this.

——因为时间有序，它可以当排序键用（比如事件按 turn 排序）。

TUI 侧还有一个名字：`client_user_message_id`。它不是 turn 身份，而是**用来去重的**——
客户端重发同一条消息时靠它识别。两者别混。

#### d. 谁决定"开始 / 插话 / 拒绝"

第 51 步的 `start_or_steer`（`session/turn_input.rs:299`）是 Core 里唯一的判定点，
模块注释写得很明确：

> This is the one place Core decides whether submitted input starts a turn, steers an active turn,
> or is rejected.

逻辑是**先试 steer，失败再试 start**：

```rust
match session.steer_input(&mut input, ..., /*expected_turn_id*/ None, ...).await {
    Ok(turn_id) => { /* 插话成功 */ TurnInputSubmission::Steered { turn_id } }
    Err(NotSubmittedReason::NoActiveTurn) => {
        // 走到这里说明没有活跃 turn，考虑启动
        let _admission = session.services.extensions.admit_turn_start();  // 准入门
        let turn_context = settings.apply_started(session, submission_id, TurnStartKind::User).await?;
        session.spawn_task(turn_context, task_input, RegularTask::new()).await;
        TurnInputSubmission::Started { turn_id: submission_id }
    }
    Err(reason) => TurnInputSubmission::NotSubmitted { reason }
}
```

三个设计细节：

- **`steer_input` 先跑**，所以"插话"不需要额外判断活跃状态，一次调用就分流了。
- **准入门**（`admit_turn_start()`）在启动分支里。拿不到许可就是
  `NotSubmittedReason::ServerDraining`——host 正在关闭时不再接新 turn。
- **父代理委派的输入绕过准入门**（`is_delegated_input`），因为子 agent 的输入属于
  父级已有的工作，不是新会话。

协议层还提供了四种模式（`protocol/src/turn_input.rs:133`），让调用方表达意图：

| `TurnInputMode` | 语义 |
|---|---|
| `StartOrSteer` | 空闲就开新 turn，否则插话（默认） |
| `StartIfIdle` | 只在空闲时开始，否则拒绝 |
| `ContinueIfIdle { expected_previous_turn_id }` | 空闲才继续；如果这期间有别的 task 启动过就拒绝 |
| `Steer { expected_turn_id }` | 只对指定 turn 插话 |

`start_or_steer` 走到最后一步是 `spawn_task(...)`——**turn 就是在这一行诞生的**。
### ③ core 怎么处理 turn

turn 一旦诞生，`RegularTask::run` 就跑起来。从它往里看是三层嵌套：

#### a. 三层嵌套的循环

```text
RegularTask::run                     tasks/regular.rs:40
  loop {                             ← 任务级：处理"上一轮收尾后又有输入"
    run_turn                         turn.rs:163
      loop {                         ← turn 级：模型 ↔ 工具循环
        drain pending input          （steer 在这里被吸收）
        run_sampling_request         turn.rs:1618
          loop { ... }               ← 采样级：单个请求 + 流式重试
        if needs_follow_up { continue }
        else { stop hooks; return }
      }
    if !input_queue.has_pending_input { return }
    next_input = vec![]
  }
```

三层循环各自解决不同的问题，混在一起看必然迷糊：

| 循环 | 位置 | 解决什么 |
|---|---|---|
| **turn 级** | `run_turn` | **模型要求工具调用** → 执行 → 再问一次模型 |
| **采样级** | `run_sampling_request` | **同一个请求的流式重试**（网络抖动、断流） |
| **任务级** | `RegularTask::run` | **收尾之后又来了输入** → 在同一个 task 里再开一个 `run_turn` |

注意 **turn 级**和**任务级**都在处理"还有输入"，区别在时机：turn 级处理的是
**turn 进行中**到达的输入（steer），任务级处理的是 `run_turn` 刚刚收敛、
**task 尚未返回**这个极窄窗口里到达的输入。如果输入晚于 task 返回才到，两条路都不会走——
而是再走一次 ② 的产生流程，开一个全新的 turn。

**任务级**循环还有一处容易看漏：第二圈跑 `run_turn` 时传的是 `next_input = Vec::new()`
——因为新输入已经被第一圈通过 `input_queue` 吸收掉了。不写出来会以为它在重复消费输入。

#### b. turn 什么时候算结束

判据只有一行（`turn.rs:566`）：

```rust
let needs_follow_up = model_needs_follow_up || has_pending_input;
```

两个来源：

- `model_needs_follow_up` —— 来自采样结果。模型要求函数调用就为真
- `has_pending_input` —— 来自 `input_queue.has_pending_input()`，即 steer 队列非空

只有当**两者都为假**时才进入收尾分支（`turn.rs:653`）：

```rust
if !needs_follow_up {
    last_agent_message = sampling_request_last_agent_message;
    let stop_outcome = run_turn_stop_hooks(...).await;
    ...
}
```

也就是说收尾前还要跑 **Stop hook**——它有权阻断或延长。`run_turn` 的文档注释把这个语义
写得很直白：

> - If the model requests a function call, we execute it and send the output back to the model
>   in the next sampling request.
> - If the model sends only an assistant message, we record it in the conversation history and
>   consider the turn complete.

#### c. token 超限时不结束 turn，而是插队压缩

这是最反直觉的一段（`turn.rs:601`）：

```rust
let should_roll_over = needs_follow_up
    && (sess.take_new_context_window_request().await || token_limit_reached);

if should_roll_over {
    run_auto_compact(
        ...,
        InitialContextInjection::BeforeLastUserMessage { world_state, step_context },
        CompactionReason::ContextLimit,
        CompactionPhase::MidTurn,
    ).await?;
    ...
    continue;                     // ← 继续同一个 turn
}
```

三个要点：

- 触发条件是 `needs_follow_up && token_limit_reached`——**只有还需要继续时**才压缩。
  如果模型已经答完了，超限与否都不影响收尾。
- 用的是 `CompactionPhase::MidTurn` 和 `InitialContextInjection::BeforeLastUserMessage`，
  和 pre-turn 压缩的策略不同（第四幕展开）。
- `continue` 意味着**用户感知不到 turn 断过**。

#### d. turn 的形态与中断

`TaskKind` 有三个值（`state/turn.rs:68`）：`Regular` / `Review` / `Compact`。

注意 `user_shell.rs` 里那个任务（跑用户直接输入的命令，不走模型）报告的
**也是 `TaskKind::Regular`**（`tasks/user_shell.rs:76`）——所以"有四个 task 实现，
但只有三个 kind"。`SessionTask` trait 本身很小（`tasks/mod.rs:180`），只有
`kind()` / `span_name()` / `run()` / `abort()` 四个方法。

中断路径：

```text
Op::Interrupt
  -> interrupt(&sess)
    -> abort_all_tasks(TurnAbortReason::Interrupted)     tasks/mod.rs:537
```

`TurnAbortReason` 有四个值（`protocol/src/protocol.rs:4280`）：

| 值 | 什么时候用 |
|---|---|
| `Interrupted` | 用户按了中断 |
| `Replaced` | **新 task 替换旧 task**（`spawn_task` 开头就调这个，`tasks/mod.rs:278`） |
| `ReviewEnded` | 审查任务结束 |
| `BudgetLimited` | 预算耗尽 |

`Replaced` 这一条很关键：**`spawn_task` 的第一件事就是中止上一个 task**。
这保证了"一个 thread 同时只有一个 turn"这条不变量不是靠调用方自觉，而是靠机制。

（这里补一个第 1 节没提的细节：`start_task`（`tasks/mod.rs:287`）本身**不做**中止，
给已经确认空闲的调用方用，例如 `maybe_start_turn_for_pending_work_with_sub_id`（`:452`）。
两条路合起来才维持住"最多一个"。）

任务的执行形态：后台 tokio task + `CancellationToken`（协作式取消）+
`AbortOnDropHandle`（兜底强杀）。优雅中断的宽限期是
`GRACEFULL_INTERRUPTION_TIMEOUT_MS = 100`（`tasks/mod.rs:71`）。

---

### ④ 取舍

**为什么允许 steer（中途插话）？代价是什么？**

因为用户经常在模型还在跑的时候想起补充条件。如果只能排队等当前 turn 结束，
要么浪费一次完整生成，要么用户被迫打断重来。

代价是状态机复杂度上了一个台阶，具体体现在三处：

- `needs_follow_up` 必须同时考虑模型意图和队列状态（`turn.rs:566`）
- 需要"任务级循环"来覆盖"收尾之后又来了输入"这种情况（`RegularTask::run`）
- 需要一个 `MailboxDeliveryPhase` 状态机来回答"这条消息还能不能塞进当前 turn"
  （`state/turn.rs:49`，注释解释了 `CurrentTurn` / `NextTurn` 的切换时机）

**为什么 turn 内允许多次模型请求，而不是一次调用就结束？**

因为工具调用必须回灌。模型输出 `function_call` 之后，执行结果要作为新的输入发回去，
模型才能继续。这不是设计选择，是工具调用范式的必然结果。

`run_turn` 的文档注释还补了一句经验观察：

> While it is possible for the model to return multiple of these items in a single sampling
> request, in practice, we generally [see] one item per sampling request.

所以实现的写法是"循环处理"，但实际路径大多是"一轮一个 item"。

**为什么 token 超限选择"压缩后继续"而不是"结束 turn"？**

因为从用户视角看，任务还没做完。如果在工具调用中途把 turn 掐掉，用户要重新描述需求，
而且之前积累的上下文可能白费。`continue` 让压缩变成**实现细节**。

代价是复杂度：压缩逻辑必须能在 turn 内部被调用，而且要处理"压缩本身也可能失败"
（`turn.rs:632` 那段 error 分支里，失败时是 `emit_turn_error_lifecycle` 然后返回 `Ok(None)`，
不是 panic）。

**为什么 `Session` 的那句注释说 "task" 而不是 "turn"？**

因为那一层描述的是**运行时在跑什么**，被管理的东西就是 `RunningTask`。而因为一个 thread
同时只允许一个 task，它等价于"同时只允许一个 turn"。更详细的读法在第 1 节 ③b。

---

## 3. 模型接入

### ① 问题

怎么跟模型说话？为什么不是一次简单的 HTTP POST？

表面答案是"发一个请求、收一个回答"，但拆开看其实是四件事：

1. **用哪个后端** —— ChatGPT 的 Codex 后端？OpenAI 官方 API？本地 Ollama？企业网关？
2. **用什么凭证** —— ChatGPT 登录态？API key？还是别的签名方式？
3. **请求长什么样** —— 走哪个协议、哪些字段、哪些是私有扩展
4. **谁真的发字节、怎么解析流式应答**

这四件事只要有一件和另一件绑在一起，换任何一样都得动其它三样。所以 Codex 把它们切成了
四层，放在四个 crate 里。这一节的主线就是先把分层理顺，再看协议和传输。

还有一个前置问题：为什么不是"一次 POST 拿一个完整回答"？因为模型生成本来就是增量的，
而首字延迟（TTFT）直接决定体感。**流式不是优化项，是基本要求。**

### ② 代码入口

第 2 节的链在第 57 步（`run_sampling_request`）接入本节——那里正是"要发一次模型请求"的时刻。
本节进入它的内部——**一个 `Prompt` 怎么变成网络上的字节，又怎么变回事件流**：

```text
57. core/src/session/turn.rs:1618             run_sampling_request          # 第 2 节的链在此接入
58. core/src/session/turn.rs:1684               try_run_sampling_request(...)
59. core/src/session/turn.rs:2572                 client_session.stream(prompt, ...)
60. core/src/client.rs:2226                    ModelClientSession::stream
61. core/src/client.rs:2237                      match wire_api + responses_websocket_enabled()
        ├─ 启用 WS  → core/src/client.rs:1842   stream_responses_websocket
        └─ 否则     → core/src/client.rs:1651   stream_responses_api        # HTTP/SSE 分支
62. core/src/client.rs:1182                          build_api_transport     -> ReqwestTransport
63. core/src/client.rs:1765                          ApiResponsesClient::new(...).stream_request(...)
64. codex-api/src/endpoint/responses.rs:61              ResponsesClient::stream_request
65. codex-api/src/endpoint/session.rs:122                 EndpointSession::stream_encoded_json_with
                                                            （套重试 + auth.apply_auth）
66. http-client/src/transport.rs:41                     HttpTransport::stream()   # ← 真正发字节
67. codex-api/src/sse/spawn_response_stream               解析 SSE 事件
68. core/src/client.rs 的 map_response_stream             变回 ResponseStream，交还第 2 节
```

两处衔接：

- **第 57 → 60 步仍在 core 内**，只是从 `session/turn.rs` 换到 `client.rs`——这条链到第 66 步才真正出网。
- **第 61 步是 §3 后面 d 小节的分岔点**：走 WS 还是 HTTP，取决于 provider 能力与会话级的 fallback 状态。

不在这条链上、但会被用到的部分：

```text
codex-rs/codex-client/src/{provider,retry,sse}.rs           # Provider 参数对象 + 重试策略
codex-rs/codex-api/src/endpoint/responses_websocket.rs      # WebSocket 推理端点
codex-rs/codex-api/src/common.rs:279                        # ResponsesApiRequest / ResponseCreateWsRequest
codex-rs/model-provider/src/{provider,auth,workspace_routing}.rs
codex-rs/core/src/client.rs:265                             # ModelClient（会话级）
codex-rs/core/src/client.rs:292                             # ModelClientSession（turn 级）
```

### ③ 关键设计

#### a. 四层分工

| 层 | 负责 | 不负责 |
|---|---|---|
| `model-provider` | 选后端、鉴权、能力声明、模型目录 | **不发推理请求** |
| `codex-api` | 构造类型化请求、解析 SSE / WS 帧 | 不管选哪个后端 |
| `codex-client` | `Provider` 参数对象、重试策略 | 不认识端点语义 |
| `http-client` | `HttpTransport` trait + `ReqwestTransport` | 不认识 Codex |

一个容易搞错的点：**`model-provider` 不是"发请求的模块"**。它的方法全是产出配置或策略：

```rust
fn api_provider(&self)    -> Provider                 // 纯参数：base_url / headers / retry / timeout
fn api_auth(&self)        -> SharedAuthProvider       // 凭证，等着别人调用
fn capabilities(&self)    -> ProviderCapabilities     // 能力上界
fn models_manager(&self)  -> SharedModelsManager      // 模型目录
```

`Provider` 是 `codex-client` 里的纯数据结构，只会拼 URL：

```rust
pub struct Provider {
    pub name: String,
    pub base_url: String,
    pub query_params: Option<HashMap<String, String>>,
    pub headers: HeaderMap,
    pub retry: RetryConfig,
    pub stream_idle_timeout: Duration,
}

pub fn url_for_path(&self, path: &str) -> String   // 就是 base_url + "/" + path
```

推理的实际调用链（每一行都能在编辑器里跳过去）：

```text
core/src/client.rs:1765    ApiResponsesClient::new(transport, api_provider, api_auth)
                             .stream_request(request, options)
  core/src/client.rs:1182    build_api_transport            -> ReqwestTransport
  codex-api/src/endpoint/responses.rs:61                   编码请求体、挂 header
  codex-api/src/endpoint/session.rs:139                    run_with_request_telemetry：
                                                             套重试策略 + auth.apply_auth(req)
  http-client/src/transport.rs:41  HttpTransport::stream() <- 真正发字节
  codex-api/src/sse/spawn_response_stream                  解析 SSE 事件
```

注意 `EndpointSession` 承担了两件横切的事：**重试策略**（`self.provider.retry.to_policy()`）
和**鉴权注入**（真正发送前调 `auth.apply_auth(req)`）。所以"用什么凭证"由 `model-provider`
决定，**执行却发生在 `codex-api`**。

唯一的例外：`model-provider` 自己也发一次 HTTP，但只用于拉模型目录
（`models_endpoint.rs`，`/models`，5s 超时 / 1MB 上限），不是推理。

最后是两个编排对象，它们的存活范围不同：

| | 存活范围 | 持有 | 为什么 |
|---|---|---|---|
| `ModelClient` | 整个 session | auth、provider 选择、thread id、**传输 fallback 状态** | fallback 一旦触发就是会话级的 |
| `ModelClientSession` | **单个 turn** | WebSocket 连接（懒建）、上一次请求、`turn_state` sticky token | 跨 turn 复用会把上一轮的 token 带进下一轮 |

`ModelClientSession` 的注释直接警告：

> Create a fresh `ModelClientSession` for each Codex turn. Reusing it across turns would replay
> the previous turn's sticky-routing token into the next turn, which violates the client/server
> contract and can cause routing bugs.

#### b. 请求格式：Responses API + Codex 私有扩展

请求体是标准 Responses API 形状（`codex-api/src/common.rs:279`）：

```rust
pub struct ResponsesApiRequest {
    pub model: String,
    pub instructions: String,
    pub input: Vec<ResponseItem>,
    pub tools: Option<ResponsesApiTools>,
    pub tool_choice: String,
    pub parallel_tool_calls: bool,
    pub reasoning: Option<Reasoning>,
    pub store: bool,
    pub stream: bool,
    pub stream_options: Option<StreamOptions>,
    pub include: Vec<String>,
    pub service_tier: Option<String>,
    pub prompt_cache_key: Option<String>,
    pub text: Option<TextControls>,
    pub client_metadata: Option<HashMap<String, String>>,
    pub access_programs: Option<AccessPrograms>,   // <- Codex 私有
}
```

前 14 个字段基本就是 Responses API 的标准定义。Codex 在此基础上叠了四层私有内容：

| 私有叠加 | 位置 |
|---|---|
| `access_programs` 字段（cyber 访问分级） | `common.rs:31` |
| `x-codex-*` 私有 header（installation id、turn metadata、turn state……） | `core/src/client.rs:161` |
| namespace 形态的工具定义（不是扁平 function 列表） | `tools/src/responses_api.rs` |
| 私有端点（`alpha/search`、`memories/trace_summarize`） | 见下面的 e |

还有两个容易忽略的事实：

- **默认 base_url 不是 `api.openai.com`**，而是 `https://chatgpt.com/backend-api/codex`
  （`model-provider-info/src/lib.rs:77` 的 `CHATGPT_CODEX_BASE_URL`）。
- **`wire_api = "chat"` 已经不支持了**（`model-provider-info/src/lib.rs:97`）：

  ```rust
  const CHAT_WIRE_API_REMOVED_ERROR: &str =
      "`wire_api = \"chat\"` is no longer supported.\n
       How to fix: set `wire_api = \"responses\"` in your provider config.";
  ```

  也就是说接入一个自定义 provider 的门槛不是"兼容 OpenAI"，而是"兼容 Responses API"。

#### c. SSE 是什么

要讲清楚两种传输，得先知道 SSE（Server-Sent Events）是什么。

**它就是一个永不结束的 HTTP 响应。** 客户端发一次普通 GET，服务端回
`Content-Type: text/event-stream`，然后保持这个响应打开、持续往里写文本。
没有协议升级，没有握手，HTTP/1.1 就够。

线格式是纯文本，一条事件由若干 `字段: 值` 行组成，**以空行结束**。
Codex 实际解析的就是这个（`codex-api/src/sse/responses.rs`）：

```text
event: response.created
data: {"type":"response.created","response":{...}}

event: response.output_text.delta
data: {"type":"response.output_text.delta","delta":"你"}

event: response.completed
data: {"type":"response.completed",...}

```

| 字段 | 用途 |
|---|---|
| `event:` | 事件类型 |
| `data:` | 载荷（JSON），可以重复多行 |
| `id:` | 事件 ID，断线重连时用于续传 |
| `retry:` | 重连间隔（毫秒） |

三个行为特征：

- **单向**：只有服务端 → 客户端。要发东西只能另开请求（LLM 场景就是把 prompt
  放在起始请求的 body 里）
- **自带重连语义**：`id:` 给了续传依据
- **可被中间设备处理**：普通 HTTP 响应，代理 / CDN / 负载均衡都认得

`sse/responses.rs` 里还有一条硬性约定：流必须以 `response.completed`
（或 `response.incomplete`）结束，否则报错——代码里的错误信息就是
`"stream closed before response.completed"`。

**为什么 LLM 流式天然用它**：token 生成就是单向增量流，没有双向交互需求；
而 SSE 复用了整个 HTTP 生态，`curl -N` 就能调试。

#### d. 两种传输，两种状态模型

同一个 `/responses`，Codex 写了两套实现。它们的差别不只是"传输方式"，而是**状态模型**：

```rust
// HTTP 版：没有 previous_response_id
pub struct ResponsesApiRequest { ... }

// WebSocket 版：有
pub struct ResponseCreateWsRequest<'a> {
    pub previous_response_id: Option<String>,
    pub generate: Option<bool>,          // prewarm 用
    ...
}
```

全仓库搜 `previous_response_id`，**只出现在 WebSocket 路径**
（`common.rs:337` 和 `responses_websocket.rs`）。所以：

| | HTTP / SSE | WebSocket |
|---|---|---|
| 状态 | 无状态，每轮发全量 `input` | 增量，靠 `previous_response_id` 让服务端接上下文 |
| 连接 | 每次新建 | 懒建 + turn 内复用 |
| 额外能力 | — | prewarm（`generate: false`）、连接复用 |

WebSocket 的定位是**"有就赚到"的优化，不是前提假设**。三条证据：

1. 由 provider 能力门控，**默认关闭**：

   ```rust
   /// Whether this provider supports the Responses API WebSocket transport.
   #[serde(default)]
   pub supports_websockets: bool,
   ```

2. 有一条校验规则暴露了它的"非标配"程度——Bedrock（AWS 认证）不允许开启，
   因为 WS 升级握手还没实现 SigV4 签名。

3. **会话级 fallback**：一旦某轮退回 HTTP，后续 turn 都不再尝试
   （`client.rs:258` 的注释："WebSocket fallback is session-scoped"）。
   `force_http_fallback()` 会打点 `codex.transport.fallback_to_http`。

还有一处取舍写在 `client.rs` 头注释里，标题就叫 **Retry-Budget Tradeoff**：

> WebSocket prewarm is treated as the first websocket connection attempt for a turn.
> If it fails, normal stream retry/fallback logic handles recovery on the same turn.

也就是说 prewarm 和正常请求**共享重试预算**，prewarm 失败不会额外消耗一次重试机会。

#### e. 能力协商：提前关门，不是事后探测

前面提到 Codex 有几个私有端点。**它们是靠配置声明被关掉的，不是运行时探测出来的。**

第一层是能力上界（`model-provider/src/provider.rs:52`）：

```rust
/// These capabilities are a provider-owned upper bound. Callers can disable
/// more functionality through normal config, but should not expose a feature
/// that the active provider marks unsupported here.
pub struct ProviderCapabilities {
    pub namespace_tools: bool,
    pub image_generation: bool,
    pub web_search: bool,
    pub external_web_access: bool,
    pub remote_compaction: RemoteCompactionSupport,
}
```

注释把语义说死了：**这是上界，不是探测结果**。调用方只能往下关，不能往上开。

第二层是"这是不是 Codex 自己的后端"：

```rust
pub fn supports_codex_backend_routes(&self) -> bool {
    self.is_openai()
        && self.base_url.as_deref().is_none_or(|base_url| {
            base_url.trim_end_matches('/').ends_with("/backend-api/codex")
        })
}
```

要求 provider 名是 `openai` **且** base_url 指向 ChatGPT 的 Codex 后端。
换成自定义 provider 直接返回 `false`。

第三层是"在装配期就把工具摘掉"。以 web search 为例
（`ext/web-search/src/extension.rs:47`）：

```rust
available: (config.model_provider.is_openai()
    || config.model_provider.uses_openai_actor_authorization()
    || config.model_provider.supports_standalone_web_search)
    && web_search_mode != WebSearchMode::Disabled,
```

不满足时，工具**根本不会出现在模型可见的工具表里**——不是"调用之后收 404"。

好处和代价都很明确：模型不会去调一个不存在的工具；但如果某个 provider 其实支持
却没声明，就白白用不上，得手工在 config.toml 里打开标志位。

另外两件相关的事：

- 私有端点的判定走的是同一套（`alpha/search`、`memories/trace_summarize`）。
  注意后者**当前没有生产调用点**，记忆抽取实际走主模型的 Phase 1 / Phase 2。
- sticky routing 用的是另一个 header：`x-codex-turn-state`；工作区路由用的是
  `x-openai-account-routing-override`（`model-provider/src/workspace_routing.rs:12`）。
  请求元数据 `x-codex-turn-metadata` 的三种含义见 [第零幕](act-00-prerequisites.md)。

### ④ 取舍

**为什么"配置 / 协议 / 传输"必须分成三层？**

因为它们的变更频率和变更原因完全不同。换后端只动 `model-provider`；换协议格式只动
`codex-api`；换传输实现（测试用 mock、或者走代理）只动 `http-client`。
`EndpointSession<T: HttpTransport>` 是泛型的，所以测试里能直接塞 `CapturingTransport`
——`codex-api` 里大量测试就是这么写的。

**假设"服务端有状态"换来了什么？**

换来的是 `previous_response_id` 增量和 prewarm，省带宽也省 prompt 处理。
代价是完全依赖对方实现——自托管 / 离线场景会退化成 HTTP 全量模式，
或者只能用本地模型（而 `wire_api` 那一关已经把非 Responses 后端挡在外面了）。

**为什么 turn 级状态显式传给 streaming 方法，而不存在 client 上？**

为了让 turn 的存活范围在**调用点**可见。`ModelClient` 是会话级的，如果它偷偷缓存了
model / reasoning / turn_state，那"这一轮用的是哪个模型"就变成了隐式状态——出错时极难排查。
显式传参的代价是方法签名变长，收益是可读性和可测试性。

**为什么能力是声明式上界，而不是运行时探测？**

探测要多一次失败往返，而且失败本身可能有副作用（计费、限流、错误日志）。
声明式的代价是"声明错了就白白用不上"，需要靠配置覆盖来兜底。
Codex 选的是前者，和它整体"不探测、只声明"的取向一致。

---

## 4. 上下文组装

### ① 问题

模型每轮到底看到什么？这些东西从哪来、谁负责大小？

这一节的关键认知是**反直觉**的：**上下文组装发生在"记录"的时候，不在"发送"的时候。**
发送请求时只是从历史里取一份快照。想明白这一点，整条链就通了。

先把两个词说清楚，否则后面会一直绕：

- **历史** —— 代码里就是 `ContextManager`（`core/src/context_manager/history.rs:89`，注释原文是
  *"Transcript of thread history"*）。它挂在 `SessionState.history` 上（`core/src/state/session.rs:75`），
  并增量落盘到该 thread 的 rollout 文件。所以**历史归属 thread，不归属 `Session`**——进程退出时
  `Session` 析构、rollout 还在，`codex resume` 用 `apply_rollout_reconstruction`
  （`core/src/session/mod.rs:1703`）把它重建出来，还是同一条历史。
- **上下文** —— 模型能看到的那份窗口，就是**这份历史本身**；发送时只是取它的快照，不是另组一份。

边界的问题也由此明确了：**不存在"每个 sampling request 一套上下文"这回事。** 每次请求的
`prompt.input` 都是"当前整条 thread 历史"在那一刻的快照；请求之间的差异只来自三处：

- 这一轮新写进去的 item（下面 ② 里那些"写"）；
- 发送时的模态裁剪（`for_prompt`）；
- **压缩**——唯一会真正改写边界的操作，而且是**整体替换**历史（`HistoryReplacement` /
  `replace_compacted`，`core/src/context_manager/history.rs:686`）。替换之后，模型窗口就小于
  磁盘上的完整记录了。

（另注意：模型一次请求看到的不止 `input`。`base_instructions` 和本轮工具表 `tools` 在 `Prompt` 里是
**独立字段**，不走 `input`——见 ③e / ③f。）

于是问题变成三个：

1. 一段要注入模型的上下文，是怎么从代码变成历史里的一条 item 的？
2. 为什么注入的内容都有类型和大小上限？
3. 已经注入过的内容，怎么判断它"还在不在历史里"？

第 3 个问题看起来奇怪，但它是下一节（world state）能工作的前提。

### ② 代码入口

两节的接点先定住：第 3 节的链从第 58 步起用到一个 `&prompt`——`try_run_sampling_request(..., &prompt, ...)`，
第 59 步 `client_session.stream(prompt, ...)` 把它送上网；而这个 `Prompt` 是在第 57 步
`run_sampling_request` 体内拼出来的。所以两节的接点是同一个 `Prompt`——**第 3 节讲它怎么变成
网络上的字节，本节讲它怎么被组装出来。**

先看"取"这一侧，它很短——因为真的只是取一张快照：

```text
【读】core/src/session/turn.rs:516   clone_history().for_prompt(...)    # 取一份历史快照
【发】core/src/session/turn.rs:524   run_sampling_request(...)          # 第 3 节的入口
        └ core/src/session/turn.rs:1659   prompt_input（重试 / 续圈时重取）
          → core/src/session/turn.rs:1667   build_prompt(...)  ->  Prompt
```

（`prompt_input` 只有两个来源：首圈用 `run_turn` 拼好的 `sampling_request_input`；重试 / 续圈才走
`clone_history().for_prompt(...)`。）

"组装"不在这里发生——它**没有独立的阶段**，就发生在"往历史里写 item"的那一刻。写历史的唯一
入口是：

```text
core/src/session/mod.rs:3461     record_conversation_items    # 写历史的唯一入口
```

而它的触发**两个来源都有**：既在 `run_turn` 内部，也有 `run_turn` 之外的独立入口。

| 来源 | 在 `run_turn` 内？ | 典型场景 |
|---|---|---|
| `run_turn` 的每一圈 | ✅ | 注入项、steer 输入、world state 差异、模型回流 |
| `run_turn` 之外的独立入口 | ❌ | 会话建立期注入、异步注入、异步 hook、目标更新 |

下面按**时间顺序**排。

#### 一、turn 之外（独立触发，不经过 `run_turn`）

```text
会话建立期（新会话 / 压缩后全量重建）
  core/src/session/mod.rs:4224     build_initial_context_with_world_state
  core/src/session/mod.rs:4192     build_turn_context_contribution_items   # turn 级设置变了才注入

运行期异步送到，不等 turn 发起
  core/src/session/inject.rs:170   inject_no_new_turn        # 没有活跃 turn 就直接记录
  core/src/session/inject.rs:17    inject_if_running         # 有 turn 在跑 → 塞给当前 turn
  core/src/hook_runtime.rs:849     record_additional_contexts    # 异步 hook 完成后补的上下文
  core/src/session/retained_context.rs:80   record_user_goal_update   # 目标更新，不新开 turn
```

#### 二、turn 之内：`run_turn` 的每一圈（主路径）

进 `loop`（`core/src/session/turn.rs:426`）之前先做一次准备，然后每圈再重复一遍。所有这些"写"
都落到同一个入口 `record_conversation_items`（`core/src/session/mod.rs:3461`）——它是历史唯一的
写入口。按发生顺序：

**进 `loop` 之前（只做一次）**

```text
core/src/session/turn.rs:368   run_hooks_and_record_inputs(TurnStart)
                               跑输入类 hook（可以阻断本轮）+ 把本轮自己的 input 记进历史
core/src/session/turn.rs:399   注入项逐条记录
                               skills / plugins 的 injection_items
```

**`loop` 每圈（顺序固定，都发生在上面那张快照之前）**

```text
core/src/session/turn.rs:439   run_hooks_and_record_inputs(SteeredUserInput)
                               排空本圈的 steer / 排队输入：先过 hook，再记进历史
                                 ↳ turn.rs:866 → :902 record_pending_input（hook_runtime.rs:716）
                                   → session/inject.rs:123 record_annotated_conversation_items
core/src/session/turn.rs:454   rollout_budget::maybe_record_reminder                  # 预算提醒
core/src/session/turn.rs:500   time_reminder::maybe_record_current_time_reminder      # 时间提醒
core/src/session/turn.rs:507   record_step_world_state_if_changed                     # world state 差异
core/src/session/turn.rs:512   record_reasoning_effort_override                       # 推理强度覆盖
```

**模型回流时（在【发】之后）**

```text
core/src/stream_events_utils.rs:100   record_conversation_items 的落点
  ↑ :93    record_completed_response_item_with_finalized_facts
  ↑ :315   handle_output_item_done            （← core/src/session/turn.rs:2780）
  ↑ core/src/session/turn.rs:2418  handle_assistant_item_done_in_plan_mode  （← :2734，plan mode）
core/src/stream_events_utils.rs:416   工具错误回灌的文本
```

（`core/src/session/turn.rs:604` 的 `token_budget::maybe_record` 也在这一档——它在**采样之后**触发。）

#### 三、其它任务形态（也写历史，只是不走 `run_turn`）

```text
core/src/tasks/review.rs:239      Review 任务
core/src/tasks/user_shell.rs:468  user shell 任务（不走模型）
core/src/tasks/mod.rs:975         中断收尾：写 InterruptedTurnHistoryMarker
core/src/compact.rs:361           压缩：整体替换历史，刻意跳过 record_conversation_items
```

#### 小结：这些"写"与【发】是什么关系

**没有"调用关系"。** 上面这些"写"都不是被【发】调用的，它们是【发】的"上游生产者"——发出去之前，
历史已经被前面这些步骤写完；模型返回的内容又写回历史，供下一圈取快照。落到代码上就是本节标题
那句话：**组装发生在这些"写"里（记录时），取快照那一步什么都不组装。**

至于"写进去的到底是什么东西、怎么变成一条 item"——那是 ③ 的前两节（a / b）。

### ③ 关键设计

#### a. 写进去的是什么：`ResponseItem` 与 `ContextualUserFragment`

不管从 ② 里哪个入口进，写进历史的都是 `ResponseItem`（历史本身就是 `Vec<ResponseItemEnvelope>`）。
而"注入类"内容——权限说明、world state 差异、记忆摘要、技能正文……——统一用一种形态表达。注入
模型的每一段上下文都必须实现这个 trait（`codex-rs/context-fragments/src/fragment.rs:64`）：

```rust
pub trait ContextualUserFragment {
    fn role(&self) -> &'static str;                       // "developer" / "user" / ...
    fn content_kind(&self) -> ContentItemKind;            // 稳定的 "<feature>.<name>" 分类
    fn requires_separate_message(&self) -> bool { false } // 是否必须独占一条 item
    fn markers(&self) -> (&'static str, &'static str);    // 首尾标记
    fn body(&self) -> String;
    fn type_markers() -> (&'static str, &'static str) where Self: Sized;
    fn matches_text(text: &str) -> bool where Self: Sized { ... }

    // 下面三个是 trait 提供的默认实现，实现方不用自己写
    fn render(&self) -> String { ... }                      // markers + body = 最终文本
    fn render_fragment(&self) -> RenderedFragment { ... }   // role + render() + content_kind
    fn into(self) -> ResponseItem { ... }                   // 直接变 ResponseItem
}
```

三个要点：

- **`render()` 不加分隔符**：`render()`（`fragment.rs:91`）是默认实现，做的事就是把 `markers()` 和
  `body()` 拼起来——`format!("{start_marker}{body}{end_marker}")`——中间**不插任何分隔符**。
  所以 trait 的文档注释才要求"实现方要自己在 `body()` 里带上需要的空白"。
- **`markers()` 要成对出现**：要么都有、要么都空。有标记的片段才能被 `matches_text()` 认出来
  ——这是"这段注入内容还在不在历史里"的判据，下一节会用到。
- **`into()` 直接变 `ResponseItem`**：`render_fragment()` 把 `role` + 渲染结果 +
  `content_kind` 打包，再转成 `ResponseItem::Message`。

`content_kind` 的语义在 trait 注释里写死了：*"Returns a stable `<feature>.<name>`
classification, using `generic` for shared fragments."* 它不为 UI 服务，而是给**服务端**
做分类用（对应 `Feature::ContentItemKinds`）。

这些片段都定义在哪、由谁构造（也就是 ② 里那些"写"的素材）：

```text
片段清单     codex-rs/core/src/context/mod.rs                     54 个片段模块的清单
识别注入项   core/src/context/contextual_user_message.rs
片段 → item  core/src/context_manager/updates.rs:12,32             build_rendered_message / merge_contextual_fragments（见 b）
扩展侧构造   codex-rs/ext/extension-api/src/contributors/prompt.rs  PromptFragment / PromptSlot（见 d）
```

#### b. 片段怎么合成一条 item：`merge_contextual_fragments`

有了片段形态（a），接下来看它们怎么变成一条 `ResponseItem`——规则在 `context_manager/updates.rs`：

```rust
for fragment in fragments {
    let group = if fragment.requires_separate_message() {
        MessageGroup::Standalone
    } else {
        MessageGroup::Mergeable
    };
    let rendered = fragment.render_fragment();
    // 同 role 且都可合并 -> 追加到上一条
    ...
}
```

合并的结果是一条 `ResponseItem::Message`，里面：

- `content` 是各片段的渲染结果数组
- `internal_chat_message_metadata_passthrough.content_item_kinds` 是**一一对应**的分类数组

所以"合并"只是打包成一条，**每段的身份不会丢**。

#### c. 什么时候写：三类注入时机

② 回答的是"谁在写"；这里回答"为什么有些轮次写、有些不写"——注入不是"每轮都重来"，而是分三种情况：

| 时机 | 入口 | 说明 |
|---|---|---|
| 初始上下文 | `build_initial_context_with_world_state`（`session/mod.rs:4224`） | 新会话 / 压缩后 / 需要全量重建时（= ② 的"会话建立期"） |
| world state 差异 | `record_step_world_state_if_changed`（`session/mod.rs:3607`） | 每步只发和上一份 snapshot 的差异（= ② 的 `turn.rs:507`，下一节展开） |
| turn 上下文贡献 | `build_turn_context_contribution_items`（`session/mod.rs:4192`） | turn 级设置变了才注入（= ② 的 `:4192`） |

第三种的门槛在 `session/mod.rs:4640`：

```rust
if !should_inject_full_context && turn_context_changed {
    context_items.extend(self.build_turn_context_contribution_items(step_context).await);
}
```

——**只有 `turn_context_changed` 为真才注入**。这就是"避免频繁变更造成 cache miss"那条规则
的落地：设置没变就不发，历史自然稳定，prompt cache 才能命中。

#### d. 扩展怎么往里加东西

扩展不直接构造 `ResponseItem`，而是产出 `PromptFragment`
（`ext/extension-api/src/contributors/prompt.rs`）：

```rust
pub enum PromptSlot {
    DeveloperPolicy,
    DeveloperCapabilities,
    /// Text inside the context-window message, supplied by `contribute_thread_context`.
    ContextWindow,
}
```

加三种构造函数：`PromptFragment::new` / `developer_policy` / `developer_capability`。

**注意 slot 只有三个**——扩展能决定"放哪个槽"，但**不能决定"放在历史的哪个位置"**。
位置由 Core 在组装时统一决定。这是刻意的限制，也是"不重写历史"能成立的前提。

（② 里 `turn.rs:399` 注入的 skills / plugins 注入项，以及 `build_extension_turn_input_items`
收集的扩展输入，走的就是这条路。）

#### e. "读"这一步有多简单

回到 ② 开头那条【读】。`Prompt`（`core/src/client_common.rs:23`）一共就七个字段：

```rust
pub struct Prompt {
    pub input: Vec<ResponseItem>,              // ← 就是上面写进历史的那批 item
    pub(crate) tools: Arc<[ToolSpec]>,
    pub(crate) parallel_tool_calls: bool,
    pub base_instructions: BaseInstructions,   // ← 单独一条通道
    pub output_schema: Option<Value>,
    pub output_schema_strict: bool,
    pub(crate) cyber_access_program: Option<CyberAccessProgram>,
}
```

而 `build_prompt`（定义 `turn.rs:1589`，`run_sampling_request` 里在 `turn.rs:1667` 调用）几乎是
纯赋值——它只做一件事：把 `StepContext` 里的字段搬进 `Prompt`。

真正"取上下文"发生在它上面几行（`turn.rs:1659` 的 `for_prompt`）：

```rust
let prompt_input = sess.clone_history().await
    .for_prompt(&step_context.settings.model_info.input_modalities);
```

`for_prompt`（`context_manager/history.rs:581`）的文档注释说明了它做什么：

> Returns the history prepared for sending to the model. This applies a proper normalization and
> drops un-suited items. Unsupported image and audio content is stripped from messages and tool
> outputs according to `input_modalities`.

**所以：发送 = 克隆历史 + 按模型能力裁剪。没有"拼 prompt"这一步。**

两个支撑事实：`clone_history()`（`core/src/session/mod.rs:4475` → `core/src/state/session.rs:169`）
就是 `self.history.clone()`，而 `ContextManager` 内部是 `Arc<Vec<ResponseItemEnvelope>>`——
**快照是共享的，不深拷贝**；`history_version`（`core/src/context_manager/history.rs:102`）在压缩 /
回滚这类"历史被重写"的操作时递增，用来让下游知道窗口换了。

#### f. base instructions 走的是另一条路

它**不在 `input` 里**，而是 `Prompt` 的独立字段。来源是
`get_prompt_base_instructions`（`session/mod.rs:1491`），里面有一层特殊处理：

```rust
if !config.update_plan_enabled
    && config.model_catalog.is_none()
    && matches!(instructions.provenance, Some(BaseInstructionsProvenance::Model { .. }))
{
    BaseInstructions {
        text: codex_prompts::without_update_plan_instructions(&instructions.text),
        ..instructions
    }
} else {
    instructions
}
```

——**请求时用的副本会被裁剪，但持久化 / 被 fork 继承的那份不动**。注释写得很清楚：
*"Render the request copy without changing instructions persisted or inherited by forks."*

这又是"持久状态"和"本次请求视图"分离的同一个模式（对比 §2 的 `StepContext`）。

#### g. 大小控制的两层

| 层 | 手段 |
|---|---|
| 片段自身 | 实现方在 `body()` 里截断（例：`MemoryContextFragment` 用 `TruncationPolicy::Bytes(8_900)`） |
| 整个请求 | `for_prompt` 按 `input_modalities` 裁掉模型不支持的图片 / 音频；总量由压缩负责（第四幕） |

AGENTS.md 里那几条硬规则可以对着看：

> - No history rewrite - the context must be built up incrementally.
> - Avoid frequent changes to context that cause cache misses.
> - No unbounded items - everything injected in the model context must have a bounded size and a hard cap.
> - No items larger than 10K tokens.
> - All injected fragments must be defined as structs in `core/context` and implement ContextualUserFragment trait

最后一条尤其值得注意：**"必须是 `core/context` 下的 struct"**——这就是为什么那个目录有 54 个模块，
而不是一堆散落的字符串拼接。

### ④ 取舍

三个取舍分别对应上面的设计：**类型化片段**（a / b）、**不重写历史**（② 与 c）、
**base instructions 单独通道**（f）。

**为什么不用一个大字符串模板？（对应 ③a / ③b）**

因为模板能表达"内容"，但表达不了"这段内容是什么、有多长、还在不在历史里"。
类型化片段换来四件事：可测试（每个片段单独 snapshot）、可截断（各自有上限）、
可标记（`matches_text`）、可增量（只发差异）。

代价是样板代码：加一个片段要新建模块、实现 trait、注册进 `context/mod.rs`。
从 54 个模块这个数字就能看出这个成本是真实存在的。

**"不重写历史"这条规则挡住了什么？（对应 ② 与 ③c）**

它挡住了很多看起来更简单的实现，最典型的是"每轮重新拼整个 prompt"。那种做法下，
任何一处改动都会让整个前缀变化，prompt cache 直接失效，压缩策略也没法做增量。

它带来的约束是：**新上下文只能追加，不能插入或改写**。所以像"更新一条权限说明"这种事，
只能靠下一节的 world state 差异机制实现——发一条新的差异 item，而不是去改旧的。

**base instructions 为什么要单独一条通道？（对应 ③f）**

因为它的生命周期和普通上下文不同：由配置决定、可能被 fork 继承、还有 provenance
（来自模型还是配置）。把它塞进 `input` 会让"这条指令从哪来"变得不可追溯。

代价是它不参与 `for_prompt` 的裁剪逻辑，也不能被片段机制统一管理。

---

## 5. world state 与增量渲染

### ① 问题

上一节最后落到一句话：**上下文组装发生在"写历史"的时候**。`run_turn` 每圈会写好几样东西，
其中最容易漏掉、也最特殊的一样，就是本节的主角——world state（模型可见的运行状态）。

它特殊在哪？看它装的东西：权限、工具表、插件、环境、AGENTS.md、协作模式……这些**每轮都可能变**。
而历史只能追加（§4 的结论），所以：

> 每往历史里追加一条新内容，从那条往后，所有前缀都变了。

模型 API 按输入 token 计费，服务端还有 **prompt cache**：只有前缀和上次一样才能命中。如果每轮
把"当前权限、工具表、插件、环境、AGENTS.md……"整套重发一遍，那么每一轮都会追加一大批新 item
——上一轮缓存的前缀全部作废，而且每轮都要为这些重复内容付费。

所以问题变成：**状态每轮都在变，怎么做到"只发变化的部分"？**

答案就是 `WorldState`：把状态按"类"切成 section（权限、工具表、环境、AGENTS.md……各算一个），
每个 section 有稳定 ID，每轮只渲染和上一份快照的**差异**。

### ② 代码入口

它不是独立启动的模块，而是 §4 那张【写】清单里的一行。回到 `run_turn` 的 `loop`
（`core/src/session/turn.rs:426`），在【读】之前：

```text
core/src/session/turn.rs:507
  world_state = sess.record_step_world_state_if_changed(&world_state, step_context).await?;
```

把这一行展开，就是本节的全部内容（仍然是 `--no-daemon` 那条单进程调用链上的函数调用）：

```text
core/src/session/turn.rs:507           每圈采样请求之前调用一次
  └─ core/src/session/mod.rs:3607      record_step_world_state_if_changed
       ├─ 采集   core/src/session/world_state.rs:40     build_world_state_for_step
       │          ├─ 17 个内置 section
       │          │    core/src/context/world_state/*.rs（16 个）
       │          │    + core/src/context/token_budget_context.rs
       │          └─ extensions 的 context_contributors()（扩展 section）
       ├─ 对比   core/src/context/world_state/mod.rs:415  render_history_diff
       │          └─ 每个 section 自己的 render_diff（见 ③c）
       ├─ 写历史 core/src/session/mod.rs:3461            record_conversation_items（给模型）
       ├─ 换基准 core/src/context_manager/history.rs:465  set_world_state_baseline
       └─ 写盘   RolloutItem::WorldState                   persist_rollout_items（给 resume）
```

**生命周期**：这里必须区分两个东西——`run_turn` 手里的 **live 值**，和历史里记的 **baseline**。

| 东西 | 谁持有 | 活多久 |
|---|---|---|
| `Arc<WorldState>`（live 值） | `run_turn` 的局部变量 `world_state` | **turn 内**：282 行建、每圈 507 行换、`run_turn` 返回即销毁 |
| `WorldStateSnapshot`（baseline） | `SessionState.history.world_state_baseline` | **跨 step、跨 turn**；`resume` 时从 rollout 重建 |
| `RolloutItem::WorldState`（full / patch） | rollout 文件 | **跨进程** |

代码里有一句注释把这个分工说得很直白（`codex-rs/core/src/session/mod.rs:3633`）：

> ContextManager remembers this for later turns; run_turn owns the live value.

具体时间线：

```text
run_turn 开始
  ├─ 282  world_state = record_context_updates_and_set_reference_context_item(first_step)
  │        └─ build_world_state_for_step(...) -> Arc<WorldState>
  │           需要全量重建时（reference context item 为空）写 full snapshot；
  │           否则和 history baseline 比较，有变化才写 patch
  ├─ 308  let mut world_state = world_state?;        ← turn 本地值诞生
  ├─ 356  （仅 guardian 压缩后）重建一次：历史被重置了
  ├─ loop 每圈
  │    └─ 507  world_state = record_step_world_state_if_changed(&world_state, step)
  │             └─ 新建一个 Arc<WorldState>，和上一圈的 live 值比较；
  │                写历史、更新 baseline、写 patch；旧的 Arc 在这里失去最后一个引用
  ├─ 620/783  需要压缩时 Arc::clone(&world_state) 交给 run_auto_compact
  └─ run_turn 返回 → 局部 world_state 销毁
                     baseline 留在 history 里，供下一个 turn 用；
                     full / patch 留在 rollout 里，供 resume 用
```

**时机**：每圈采样请求之前都会跑一次；但**只有差异非空时才真的写**（见 ③c 的 `None`）。
所以它是"按步发生、按需落地"，不是启动时初始化一次的东西。

**结论**：`WorldState` 对象是 **turn 级临时值**；它体现的"状态"是 **thread 级持久数据**
（history 里的 baseline + rollout 里的 full/patch）。下一个 turn 开始时，`run_turn` 会重新
build 一个 `WorldState`，再和持久化的 baseline 比较——而不是把上一轮的 `Arc<WorldState>` 直接传进来。

### ③ 关键设计

#### a. 放大这一步：build → diff → record / persist

`record_step_world_state_if_changed`（`core/src/session/mod.rs:3607`）的全文就是三段：

```rust
pub(crate) async fn record_step_world_state_if_changed(&self, previous_world_state, step_context) {
    let world_state = Arc::new(self.build_world_state_for_step(step_context).await?);   // ① 采集
    let previous_snapshot   = previous_world_state.snapshot();
    let world_state_snapshot = world_state.snapshot();

    // 给 rollout 用的差异补丁
    let world_state_item = world_state_snapshot
        .merge_patch_from(&previous_snapshot)
        .map(WorldStateItem::patch);

    // 给模型看的差异片段
    let items = merge_contextual_fragments(world_state.render_history_diff(
        Some(&previous_snapshot),
        self.state.lock().await.history.raw_items(),
    ));

    if !items.is_empty() {
        self.record_conversation_items(turn_context, model_info, &items).await;
    }
    self.state.lock().await.history.set_world_state_baseline(world_state_snapshot);
    if let Some(world_state_item) = world_state_item {
        self.persist_rollout_items(&[RolloutItem::WorldState(world_state_item)]).await;
    }
}
```

注意这里**同时产生了两个东西**，它们服务于两个不同的消费者：

| 产物 | 消费者 | 形态 |
|---|---|---|
| `items` → `record_conversation_items` | **模型** | 渲染好的文本片段，追加进历史 |
| `world_state_item` → `persist_rollout_items` | **resume / 重启** | JSON merge patch，写进 rollout |

这是本节最容易被忽略的一点：**模型看的是文字，rollout 存的是结构化数据。** 两者都是从同一对
snapshot 推出来的，所以不会各说各话。

#### b. 采集：`build_world_state_for_step` 把 section 装进 `WorldState`

先说清楚 **section** 是什么：它不是通用容器，而是"一类模型可见状态"的实现单元——权限、工具表、
环境、AGENTS.md……每一种都实现一次 `WorldStateSection` trait（见 ③c），声明自己的稳定 ID 和
比较用的 Snapshot 类型。`WorldState` 本身只是一个有序容器：

```rust
pub(crate) struct WorldState {
    sections: IndexMap<&'static str, Box<dyn ErasedWorldStateSection>>,
}
```

`IndexMap` 保序，所以 section 的排列顺序是确定的、有语义的（这一节后面会看到为什么顺序不能乱）。

`build_world_state_for_step`（`core/src/session/world_state.rs:40`）把这一圈的 17 个内置 section 逐个
`add_section(...)` 进 `WorldState`。开头先重现了一遍 §4 ③f 的裁剪规则：

```rust
let model_instructions = render_model_instructions(model_info);
let model_instructions = if !turn_context.config.update_plan_enabled
    && turn_context.config.model_catalog.is_none()
    && (turn_context.config.base_instructions.is_none()
        || matches!(turn_context.config.base_instructions_provenance,
                    Some(BaseInstructionsProvenance::Model { .. })))
{
    codex_prompts::without_update_plan_instructions(&model_instructions)
} else {
    model_instructions
};
let base_instructions = self.get_prompt_base_instructions().await.text;
```

——和 §4 里 `get_prompt_base_instructions` 是同一条规则，两处都要用，所以两处都写了一遍。

**顺序即语义**：section 在历史里的排列顺序会影响模型的理解，所以装配顺序是写死的，不是随手插。
扩展 section 走 `add_extension_section`（`core/src/context/world_state/mod.rs:367`），其中 `host_skills` 被硬编码插到
`permissions` 前面：

```rust
if id == "host_skills" && let Some(index) = self.sections.get_index_of(PermissionsState::ID) {
    self.sections.shift_insert(index, id, section);   // ← 插到 permissions 前面
} else {
    self.sections.insert(id, section);
}
```

#### c. 每个 section 自己决定怎么渲染差异

采集只是把状态摆好。真正决定"这一轮要不要告诉模型"的，是每个 section 自己的 `render_diff`。
trait 定义在 `core/src/context/world_state/mod.rs:226`：

```rust
pub(crate) trait WorldStateSection: Send + Sync + 'static {
    const ID: &'static str;                        // 稳定 ID，会写进 rollout
    type Snapshot: DeserializeOwned + Serialize;   // 只装"比较需要"的数据

    fn snapshot(&self) -> Self::Snapshot;

    /// Whether the section contributes comparison state to persisted rollouts.
    fn should_persist(&self) -> bool { true }

    fn render_diff(
        &self,
        previous: PreviousSectionState<'_, Self::Snapshot>,
    ) -> Option<Box<dyn ContextualUserFragment>>;   // None = 这轮不用告诉模型任何事
}
```

三条契约写在注释里，值得逐条读：

> `ID` is persisted in rollouts and must remain stable.
> `Snapshot` should contain only the comparison data needed to decide what the model must be told
> next, and must not serialize to null because merge-patch nulls represent deletion.

也就是说：

- **`ID` 是跨版本的契约**，改了会让旧 rollout 对不上。
- **`Snapshot` 不是状态本身，是"比较用的数据"**。它可以比真实状态小得多。
- **不能序列化成 `null`**，因为 RFC 7386 里 null 表示"删除"。

`render_diff` 的返回值是 `Option`——**返回 `None` 表示"这轮这片状态没变，不需要发"**。
这就是"增量"的实现方式：不是发一个空的差异，而是压根不产生 item。

看一个真实例子（`core/src/context/world_state/permissions.rs:94`）：

```rust
fn render_diff(&self, previous: PreviousSectionState<'_, Self::Snapshot>)
    -> Option<Box<dyn ContextualUserFragment>>
{
    match (previous, &self.snapshot) {
        (PreviousSectionState::Known(PermissionsSnapshot::Current {
            instructions: previous_instructions,
            approved_command_prefixes: previous_prefixes,
        }), PermissionsSnapshot::Current { instructions, approved_command_prefixes }) => {
            // 逐字段比较，只把变了的部分渲染出来
        }
        ...
    }
}
```

——每个 section 的差异逻辑都是**手写的**，没有通用框架。这是刻意的：只有它自己知道
"哪部分变了才值得告诉模型"。

#### d. 三态：为什么需要 `Unknown`

比较的基准不总是存在。`PreviousSectionState`（`core/src/context/world_state/mod.rs:208`）把情况分成三态：

```rust
pub(crate) enum PreviousSectionState<'a, T> {
    /// No persisted snapshot or matching fragment exists in retained history.
    Absent,
    /// Retained history contains the section, but its typed snapshot is unavailable.
    Unknown,
    /// The exact persisted snapshot is available.
    Known(&'a T),
}
```

三种情况对应的处理完全不同：

| 状态 | 含义 | 后果 |
|---|---|---|
| `Absent` | 从没发过 / 已经被压缩丢掉 | 走 `render_full`，全量发一次 |
| `Known` | 有精确的上一份 snapshot | 走真正的差异渲染 |
| `Unknown` | **历史里有这段内容，但拿不到结构化 snapshot** | 见下 |

`Unknown` 是给**旧 rollout** 准备的。判据在 `render_history_diff`（`core/src/context/world_state/mod.rs:415`）：

```rust
} else if has_legacy_fragment(items.clone(), section) {
    PreviousSectionState::Unknown
} else {
    PreviousSectionState::Absent
}
```

如果 snapshot 丢了（比如历史被压缩），但扫描保留的历史 item 还能认出这条 section 曾经发过
（靠 §4 讲的 `markers()` / `matches_legacy_fragment`），那就标成 `Unknown`。

**`Unknown` 的意义**：既不能当作"没发过"去全量重发（会重复），也不能当作"已知"去做差异
（没有基准）。所以 section 必须自己处理这一态——通常是"按当前状态全量发一次，重建基准"。

还有第五种情况藏在上面的代码里（`core/src/context/world_state/mod.rs:417`）：

```rust
if section.has_retained_fragment_matcher()
    && !has_retained_fragment(items.clone(), section)
{
    PreviousSectionState::Absent          // ← snapshot 说发过，但历史里找不到
}
```

——**snapshot 说有，但历史里已经没了**（被压缩擦掉）。这种情况必须降级成 `Absent`，
否则模型会以为自己看过这段内容，而实际上没有。这是"snapshot 和历史可能不同步"的兜底。

#### e. 第二轨：快照与 RFC 7386 merge patch

给模型的那一轨（③a 的 `items`）是文字；给 resume 的那一轨需要一个可增量、可回放的结构。
`WorldStateSnapshot` 就是后者：

```rust
pub(crate) struct WorldStateSnapshot {
    sections: BTreeMap<String, Value>,     // section ID -> 该 section 的 snapshot
}
```

它要解决的是"怎么把这份快照存进 rollout，且增量存储"。

先把名词解释掉：**RFC 7386 就是 "JSON Merge Patch" 的标准**——用一段局部 JSON 描述"如何把旧文档
改成新文档"：对象里出现的字段覆盖旧值，值为 `null` 表示删除该字段，数组则整体替换。用它来存状态，
就不必每步存整份 snapshot，只存"从上一份到这一份的变化"。示意：

```text
上一份            当前                 merge patch
{"a": 1, "b": 2}  ->  {"a": 1, "c": 3}  ->  {"b": null, "c": 3}
```

对应到代码就是这两个函数：

```rust
pub(crate) fn merge_patch_from(&self, previous: &Self) -> Option<Map<String, Value>> { ... }  // 算 patch
pub(crate) fn apply_merge_patch(&mut self, patch: &Map<String, Value>) { ... }                // 回放 patch
```

两个实现细节值得注意：

- **发删除在前**，注释写的是 *"Emit removals first to preserve insertion-ordered JSON patch
  output."* ——因为 `BTreeMap` 有序，输出是确定性的，便于测试和对比。
- **数组整体替换**。`remove_null_object_fields` 的注释：
  *"RFC 7386 reserves object-valued nulls for deletion, but arrays are replaced whole."*

回到 ③a 的双轨设计：`merge_patch_from` 的结果既是**写进 rollout 的补丁**
（`WorldStateItem::patch`），而给模型看的那一侧由 `render_history_diff` 用同一对 snapshot 生成。
同一对 snapshot，两个消费者。

**特例：`WorldStateHash`**（`core/src/context/world_state/mod.rs:265`）是个 SHA-1 指纹，基于 `role()` + `render()` 计算：

```rust
pub(crate) struct WorldStateHash(String);

impl WorldStateHash {
    pub(crate) fn from_fragment(fragment: &(impl ContextualUserFragment + ?Sized)) -> Self { ... }
}
```

它不是全局的去重机制，而是**给某些 section 当 `Snapshot` 类型用**。看
`core/src/context/world_state/multi_agent_usage_hint.rs:20`：

```rust
type Snapshot = WorldStateHash;
```

——这类 section 的"有意义的状态"就是渲染出来的那段文字。存原文浪费，存哈希就够：
对比时只要哈希相同，就说明这轮不用发。代价是**无法从 patch 反推内容**——所以这类 section 在
resume 时只能靠历史里的片段重建，这正是 `Unknown` 存在的理由之一。

#### f. 恢复时怎么用：resume 回放 patch

rollout 里存的是"full snapshot + 一串 patch"。`resume` 时按时间顺序回放
（`core/src/session/rollout_reconstruction.rs:485–500`）：

```rust
world_state_replay.reverse();                       // 收集时是倒序的，先翻回正序
let mut world_state_baseline: Option<WorldStateSnapshot> = None;
for item in world_state_replay {
    match item {
        RolloutItem::Compacted(_) => world_state_baseline = None,          // 压缩：基准作废
        RolloutItem::WorldState(ws) if ws.full => {
            world_state_baseline = Some(WorldStateSnapshot::from(&ws.state));
        }
        RolloutItem::WorldState(ws) => {
            let Some(baseline) = world_state_baseline.as_mut() else {
                tracing::warn!("ignored world-state patch without a full snapshot");
                continue;
            };
            baseline.apply_merge_patch(&ws.state);
        }
        ...
    }
}
```

三个要点：

- 必须先有 full snapshot 才能应用 patch；孤立的 patch 会被丢弃并告警。
- 压缩会让基准作废（`Compacted` → `None`），之后的 patch 要等下一份 full snapshot。
- 回放出来的 `world_state_baseline` 交给下一轮 `record_step_world_state_if_changed` 当
  `previous`——于是"模型上次看到的是哪一版"在 resume 之后仍然可比较。

#### g. 扩展怎么加 section

扩展不直接往 `WorldState` 里塞东西，而是实现 `ContextContributor` 的两个接口
（`ext/extension-api`）：

```rust
// 贡献 section
fn contribute_world_state<'a>(&'a self, input: WorldStateContributionInput<'a>)
    -> ExtensionFuture<'a, Vec<WorldStateSectionContribution>>;

// 压缩后保留部分元数据
fn retain_world_state_after_compaction(
    &self,
    previous_world_state: &serde_json::Map<String, Value>,
) -> serde_json::Map<String, Value> { serde_json::Map::new() }
```

第二个的注释解释得很清楚：

> Return only this contributor's section IDs, without rendered text or availability state.
> Core persists these partial sections so the next step can rebuild full context without losing
> decisions that are independent of model-visible history.

也就是：压缩会丢掉渲染文本，但扩展可以把自己 section 的**决策数据**保留下来，
这样下一轮不用重新做一遍判断。

采集时这些贡献由 `build_world_state_for_step` 统一收进 `WorldState`（③b 的
`add_extension_section`），所以扩展的 section 和内置 section 走的是同一条 diff / record 路径。

### ④ 取舍

**增量渲染省的是 prompt cache 的钱，代价是什么？**

省下来的很具体：每轮只追加真正变化的部分，前缀保持稳定，缓存能命中；同时输入 token 数也降下来。

代价同样具体，集中在三处：

1. **每个 section 都要手写差异逻辑**（17 个内置 + 扩展），没有通用框架。
2. **多了一份必须保持同步的状态**：rollout 里的 snapshot 和模型历史里的文本。
   上面那两处兜底（`Unknown`、以及"snapshot 说有但历史里找不到就降级成 `Absent`"）
   就是在处理两者不同步的后果。
3. **`ID` 和 `Snapshot` 的形态变成了跨版本契约**——改它们要考虑旧 rollout 怎么读。

**为什么 `Unknown` 这个中间态是必要的？**

因为它对应一种真实且无法回避的情况：**历史里有这段内容，但结构化基准已经没了。**

典型场景是压缩——压缩会丢掉渲染文本，也可能让某条 section 的 snapshot 失去对应关系。
这时如果判成 `Absent`，就会全量重发一次（重复注入）；如果硬当 `Known` 处理，又没有基准可比。

所以 `Unknown` 的语义是"**我知道模型看过，但不知道我看到的是哪一版**"，
处理方式由 section 自己决定——通常是以当前状态重建基准。

**为什么模型看文本、rollout 存 patch，而不是统一成一种？**

因为两者的需求相反：

- 模型需要**自然语言**（它读的是 message）
- resume 需要**结构化、可增量、体积可控**的数据

统一成文本的话，resume 时就得反过来解析自然语言；统一成 JSON 的话，模型看到的就是
一堆结构化数据，那不是它消费的形态。

代价是这个双轨必须**从同一对 snapshot 推出来**（代码里正是这么做的：同一个
`previous` / `current` 分别喂给 `render_history_diff` 和 `merge_patch_from`），
否则两边会不一致。

**这一节和 §4 是什么关系？** 一句话：§4 说"组装发生在写的时候"，本节就是那个"写"里最大的一块。
它每圈执行一次（`core/src/session/turn.rs:507`），但只有状态真的变了，才会在历史里留下一条 item、在 rollout 里
留下一条 patch。

---

## 幕末收获

读完这一幕，你应该能：

- 说清 turn / thread / session 三者的边界
- 画出"用户输入 → prompt → 模型 → 回复"的完整数据结构流转
- 解释 Codex 为什么把上下文管理当成一等公民，而不是拼字符串

## 下一幕引子

到这里模型只会说话。下一幕解决"让模型能动手"——
工具怎么抽象、循环怎么终止、以及模型要跑 `rm -rf` 时谁来拦。

→ [第二幕：工具循环](act-02-tool-loop.md)
