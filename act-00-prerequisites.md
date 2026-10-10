# 第零幕：预备知识 —— 进程、结构与术语

> **幕目标**：在读任何代码之前，建立进程地图、认识核心结构、掌握设计约定与前置概念。
> 读完这一幕，第一幕的调用链才不会看起来像在同一个进程里绕圈。
>
> **前置依赖**：无。这一幕是后面四幕的地基。
>
> **状态**：正文已完成。

---

## 目录

- [进程与角色](#进程与角色)
- [基础结构：先认识这些名字](#基础结构先认识这些名字)
- [设计原则与前置知识](#设计原则与前置知识)

---

这一节是给后面所有内容打地基。分三部分：

1. **进程与角色** —— 这个程序由几个进程组成、app-server 为什么存在
2. **基础结构** —— `ThreadManager` / `CodexThread` / `Session` / `SessionSource` 这些名字分别代表什么
3. **设计原则与前置知识** —— 从这些结构里能读出来的约定，以及读代码需要的前置概念

### 进程与角色

在钻任何函数之前，先建立一张进程地图。否则第一幕的调用链会看起来像在同一个进程里绕圈。

**一句话**：Codex 不是"客户端 + 服务端"，也不是"一个进程干所有事"。
它是**一个主二进制 + 若干按需拉起的辅助进程**，而 app-server 是这个结构里的中枢——
它既可能是同进程的一个模块，也可能是一个独立的后台进程，**取决于启动参数**。

#### 六类进程

| # | 进程 | 何时存在 | 干什么 | 入口 |
|---|---|---|---|---|
| 1 | `codex` 主进程 | 用户启动 | CLI 分发 + 前端（TUI / exec） | `cli/src/main.rs:1013` |
| 2 | 多调用二进制副本 | 按需 fork/exec 自己 | 沙箱、apply_patch、exec helper | `arg0/src/lib.rs:60` |
| 3 | app-server | 内嵌 / daemon / 远程 | core 的接口层（JSON-RPC） | `app-server/src/in_process.rs:377` |
| 4 | exec-server | 需要远端执行环境时 | 生成与控制子进程 | `codex-rs/exec-server` |
| 5 | code-mode host | 启用 Code Mode 时 | 跑 V8 | `code-mode/src/remote_session/connection.rs:158` |
| 6 | MCP server | 配置了 MCP 时 | 提供外部工具 | `rmcp-client/src/stdio_server_launcher.rs` |

除此之外，还有工具执行本身产生的进程（shell 命令等），它们属于"被管理的子进程"，不是架构组件。

#### 1. `codex` 主进程

一个二进制同时是前端和管理工具。`Subcommand`（`cli/src/main.rs:143`）里有
`Agents` / `Exec` / `Review` / `Login` / `Mcp` / `Plugin` / `AppServer` / `Sandbox` / `Doctor` /
`Apply` / `Resume` / `Queue` / `Archive` / `Fork` ……；**不带子命令就是交互式 TUI**。

#### 2. 多调用二进制（arg0 trick）

`arg0_dispatch`（`arg0/src/lib.rs:60`）读 `argv[0]` 和 `argv[1]`，决定这次扮演什么角色：

```text
argv[0] == codex-linux-sandbox      ->  codex_linux_sandbox::run_main()
argv[0] == apply_patch              ->  codex_apply_patch::main()
argv[0] == execve 包装器            ->  shell escalation（unix）
argv[1] == windows MXC / sandbox    ->  Windows 沙箱
argv[1] == exec helper / fs helper  ->  exec-server 的辅助进程
argv[1] == core apply patch         ->  apply_patch
```

它还会建一个临时目录塞进 `PATH`（`Arg0PathEntryGuard`），让子进程能按别名找到这些角色，
并在调用结束后清理。

**收益**：只需要分发一个二进制，而且辅助工具和主程序**版本永远一致**——
不会出现"sandbox helper 是旧版本"这类问题。

#### 3. app-server：为什么存在

这是理解整个结构的关键。app-server 不是"另一个产品"，它是 **core 的接口层**。

三种形态（`AppServerTarget`，`tui/src/lib.rs:316`）：

| 形态 | 通信方式 | 何时出现 | 特征 |
|---|---|---|---|
| `Embedded` | 同进程函数调用 | `--no-daemon`，或被其它条件排除时 | 无 IPC 开销 |
| `LocalDaemon` | unix socket | **默认路径**（`DaemonAutoStart` 开启且未被排除） | 多前端共享 thread |
| `Remote` | WebSocket | `--remote` | 工作区在远端 |

##### 到底会选哪一个

由 `startup_orchestration.rs:484` 决定：

```rust
let auto_start_daemon = config.features.enabled(Feature::DaemonAutoStart)
    && !cli.agents_overview
    && !cli.no_daemon
    && !app_server_target.uses_remote_workspace();
```

`Feature::DaemonAutoStart` 是 `Stage::Stable` 且 `default_enabled: true`
（`features/src/lib.rs:952-956`）。也就是说**默认情况下 Codex 会去启动或复用共享 daemon**，
此时 app-server 是**独立进程**。

只有被排除时才退回 Embedded。排除条件在 `daemon_startup::exclusion`
（`tui/src/daemon_startup.rs:25`）：

```text
--no-daemon                    # 显式要求
--oss                          # 本地模型
workload identity
CODEX_EXEC_SERVER_URL          # executor 选择
--profile
-c / --enable / --disable / --search 等命令行配置覆盖
非默认的 loader overrides
```

走 daemon 那一步是 `codex_app_server_daemon::start_with_features(...)`
（`startup_orchestration.rs:534`）；成功后设置：

```rust
app_server_target = AppServerTarget::LocalDaemon {
    endpoint: RemoteAppServerEndpoint::UnixSocket { socket_path },
    allow_embedded_fallback: false,
};
```

注意 `allow_embedded_fallback: false`——自动启动路径上**连不上就报错**（提示加 `--no-daemon`），
而不是悄悄降级。会自动降级的是另一条路径：`app_server_target_for_launch`
（`tui/src/lib.rs:1013`）在发现已有 daemon socket 时给出 `allow_embedded_fallback: true`。

最终分流点在 `start_app_server`（`tui/src/lib.rs:551`）：

```rust
let connection = if matches!(target, AppServerTarget::Embedded) {
    None
} else {
    Some(app_server_connection::connect(target).await)
};
// 连不上 + allow_embedded_fallback -> *target = AppServerTarget::Embedded;
...
start_embedded_app_server(...).await.map(AppServerClient::InProcess)
```

它也可以独立启动：

```text
codex app-server --listen stdio:// | unix://PATH | ws://IP:PORT
```

（默认 `stdio://`，见 `cli/src/main.rs:565` 的 `AppServerCommand`。）

**为什么必须要有这一层**：如果 TUI 直接调 core，那 IDE 扩展就得把"怎么开会话、
怎么提交 turn、怎么处理审批"再实现一遍。有了 app-server，core 只需要认识一套 JSON-RPC
（`thread/start`、`turn/start`、`thread/read`……），所有前端都只是它的客户端。
这也是第一幕后面所有调用链都要穿过它的原因。

##### daemon 到底持有什么

一句话：**daemon 持有 `ThreadManager` 和所有 `Session`。**

证据在构造点——`ThreadManager::new` 出现在 `app-server/src/message_processor.rs:335`，
也就是 **app-server 进程内部**。所以"哪个进程跑 app-server，thread 就活在哪个进程"。

在这个基础上，daemon 承担七件事：

| # | 职责 | 依据 |
|---|---|---|
| 1 | **持有 thread 生命周期** | thread 从 start 到 stop 全在 daemon 里；TUI 断开 ≠ thread 结束 |
| 2 | **跨客户端共享 thread** | 多个前端连同一个 daemon，看到同一批会话 |
| 3 | **存活与恢复** | 关闭时写 recovery snapshot，下次启动后台冷恢复（`app-server/src/daemon_thread_recovery.rs`） |
| 4 | **多入口的统一出口** | `codex agents` / `queue` / `archive` / `delete` 都通过它操作 thread |
| 5 | **进程级共享资源** | model catalog、auth refresh、plugins/skills watcher，一个进程一份 |
| 6 | **远程控制** | `codex remote-control` 让 daemon 可被远程控制 |
| 7 | **自身生命周期管理** | `daemon start\|stop\|restart\|version`，用文件锁序列化多个 CLI 调用 |

第 3 条值得展开。`daemon_thread_recovery.rs` 的模块注释：

> Saves persistent root thread IDs and restores them through the shared internal resume path.
> Recovery uses normal cold-resume semantics without delaying readiness for runtime loading.
> **Already-loaded runtimes remain owned by their current clients.**

——daemon 重启后会把之前加载的 thread 冷恢复回来，用的是**和 resume 同一条路径**。这就是
"TUI 关了但任务还在"的实现基础。

补一个容易忽略的点：daemon 是**受管安装**的（`managed_codex_bin` / `install_lock` /
`update_loop`），所以它**可以和 CLI 跑不同版本**——这正是那套 `compatibility_warning` /
`server_features` 检查存在的原因。

##### "只能有一个 owner"：为什么 `codex queue` 会拒绝

这是整个设计里最关键、也最容易忽略的一条约束。

`codex queue` 的拒绝逻辑有两道（`tui/src/session_queue_commands.rs:31`）：

```rust
if options.cli.no_daemon && options.explicit_remote_endpoint.is_none() {
    return Err(eyre!(
        "--no-daemon cannot be used with codex queue. Queuing must discover the shared server
         to avoid writing through a separate server."
    ));
}
```

```rust
if !explicit_remote
    && app_server.uses_embedded_app_server()
    && super::maybe_probe_default_daemon_socket(codex_home.as_path()).await.is_some()
{
    return Err(eyre!(
        "cannot queue through an embedded app server while a local app-server daemon is
         running; remove configuration overrides or use --remote"
    ));
}
```

**注意第二道**：embedded 明明能跑，但只要探测到 daemon 还在，就**拒绝**。理由是"避免通过
另一个 server 写入"。

所以设计意图不是"有 daemon 更好用"，而是：**同一个 `CODEX_HOME` 下，thread 状态只能有一个
owner；当 daemon 存在时，其他形态必须让路。**

##### `--no-daemon` 到底会不会报错

结论是**不会**——但要分层看，因为拒绝发生在三个不同的地方。

**第一层：启动期不拦。** flag 的文档就是这么写的（`tui/src/cli.rs:83`）：

```rust
/// Run without the shared background server, even if it is already running.
#[arg(long)]
pub no_daemon: bool,
```

而且代码路径**根本不探测 daemon**（`tui/src/startup_orchestration.rs:183`）：

```rust
let reuse_implicit_local_daemon = daemon_exclusion.is_none();
```

`--no-daemon` 让 `daemon_exclusion` 返回 `Some("--no-daemon")`，于是这个值是 `false`，
`app_server_target_for_launch` 直接落到 `None => AppServerTarget::Embedded`。
没有 socket 探测、没有连接尝试、没有版本比对。

**第二层：特定命令在参数校验时就拒绝。** 理由是这些命令**语义上依赖共享状态**：

```text
codex agents --no-daemon   ->  "The agents overview requires a shared server."
codex queue  --no-daemon   ->  "Queuing must discover the shared server..."
codex --no-daemon --remote ->  "--no-daemon cannot be used with --remote."（自相矛盾）
```

**第三层：真正的资源冲突由文件锁兜底。**

如果 `--no-daemon` 要接管的正是 daemon 里正在跑的那条 thread，会在写锁那一层失败
（`rollout/src/writer_lock.rs`）：

```rust
const WRITER_LOCK_DIR: &str = "thread-writer-locks";

/// Coordinates writer ownership within one Codex home.
pub struct WriterLockCoordinator { ... }

/// Acquires exclusive writer ownership, returning `WouldBlock` for an active writer.
pub fn acquire(self: &Arc<Self>, thread_id: ThreadId) -> io::Result<WriterLockGuard> {
    let path = self.directory.join(format!("{thread_id}.lock"));
    match file.try_lock() {
        Err(err) if err.kind() == io::ErrorKind::WouldBlock => { /* 有活跃 writer */ }
        ...
    }
}
```

`thread-store/src/local/mod.rs:338` 把它映射成 `ThreadStoreError::Conflict`。
也就是**每个 thread 有一把跨进程的 OS 文件锁**，落在
`$CODEX_HOME/thread-writer-locks/<thread_id>.lock`。

三种情况合起来看：

| 场景 | 结果 | 在哪一层被拦 |
|---|---|---|
| `codex --no-daemon` 起新会话 | ✅ 正常 | 不拦 |
| `codex --no-daemon resume <不活跃 thread>` | ✅ 正常 | 不拦（daemon 没持锁） |
| `codex --no-daemon resume <活跃 thread>` | ❌ Conflict | **文件锁** |
| `codex agents --no-daemon` | ❌ | **参数校验** |
| `codex queue --no-daemon` | ❌ | **参数校验** |

这个分层不是巧合：**`--no-daemon` 是"我自愿放弃共享"，文件锁是"共享状态的硬边界"**，
两者分工不同，所以行为也不同。

##### TUI 从 app-server 拿什么

TUI **从不直接调 core**，它是 app-server 的纯客户端。依赖面可以数出来——用到的
`ClientRequest` 有 80 多个，归纳成六类：

| 类别 | 代表方法 |
|---|---|
| 会话生命周期 | `ThreadStart` / `ThreadResume` / `ThreadFork` / `ThreadDelete` / `ThreadArchive` / `ThreadList` / `ThreadRead` / `ThreadSearch` |
| turn 控制 | `TurnStart` / `TurnSteer` / `TurnInterrupt` / `ThreadCompactStart` |
| **反向请求**（server → client） | `ExecCommandApproval` / `ApplyPatchApproval` / `McpElicitation` / `ToolRequestUserInput` / `DynamicToolCall` / `AttestationGenerate` / `CurrentTimeRead` |
| 配置与账号 | `ConfigRead` / `ConfigBatchWrite` / `LoginAccount` / `GetAccount` / `GetAccountRateLimits` |
| 扩展生态 | `PluginInstall` / `McpServerStatusList` / `SkillsList` / `MarketplaceUpgrade` / `HooksList` |
| 平台能力 | `FsReadFile` / `FsWriteFile` / `FsRemove` / `OneOffCommandExec` / `WindowsSandboxSetupStart` |

**第三类最需要留意**：审批、提问、MCP 授权、动态工具、证明生成、当前时间——这些是
**server 向 client 要东西**。也就是说 TUI 不只是消费者，它还是某些能力的**提供方**：
渲染审批弹窗、让用户回答问题、执行 dynamic tool、生成 attestation。

**这是双向依赖，不是单向调用。**

##### 两个只有共享 server 才能做的功能

有了 daemon 之后才可能出现两类"跨会话操作"，它们也是理解前面那些拒绝逻辑的最好例子。

**`codex queue`：往已有会话投递消息**

```bash
codex queue --thread <会话 UUID 或名字> --message "<文本>"
```

机制（`ext/queue/src/service.rs`）：

| 环节 | 实现 |
|---|---|
| 存储 | `QueueStore`（durable，落在 thread-store / SQLite） |
| 发现 | `watch_external_messages` **每 10 秒**查一次 SQLite 的 data version，只挑变化的 thread |
| 派发 | `on_thread_idle`（`ThreadLifecycleContributor`）→ `dispatch_if_idle` |
| 起 turn | `thread.start_turn_if_idle(...)` |
| 成功才删 | `Started` 才删队列项，失败留着重试 |

两个语义细节：

```rust
// 被打断时不派发 —— 中断是明确意图，不该被队列覆盖
if input.cause == ThreadIdleCause::Interrupted {
    return;
}
```

```rust
// 只接受用户输入，其它类型丢弃
if !matches!(input, TurnInput::UserInput { .. }) {
    tracing::warn!(%queued_item_id, "discarding non-user queued input");
    self.delete_locked(thread_id, queued_item_id).await?;
    continue;
}
```

它和 steer 的区别在于**时机**：

| | steer（`TurnSteer`） | queue（`codex queue`） |
|---|---|---|
| 时机 | 插进**正在跑**的 turn | 等 turn 结束、线程空闲 |
| 立刻性 | 立即 | 最多 10 秒轮询延迟 + 等空闲 |

典型场景：脚本或 CI 往一个正在运行的会话追加指令，不用重新开会话、不用 resume。

**`codex agents`：daemon 范围的会话总览**

模块注释（`tui/src/app/agents_overview.rs`）把定位写得很清楚：

> Daemon-wide overview of recent and locally retained sessions **and their subagents**.
> Tasks owned by another app server open as **frozen, read-only history snapshots**.
> Only the immediate attachment of a dashboard-created task is treated as fresh.

三件事：**所有会话 + 它们的子 agent**；别的 app-server 实例持有的任务显示为**只读快照**
（安全：不能两个 TUI 同时写）；**从面板新建的任务才真正附着**。

功能从模块名就能看出来：

| 模块 | 作用 |
|---|---|
| `agents_overview_grouping.rs` | 分组（按目录 / 状态等） |
| `agents_overview_details.rs` | 会话详情与活动 |
| `agents_overview_usage.rs` | 用量统计 |
| `agents_overview_new.rs` | **从面板新建任务**（含 worktree，`PendingWorktree`） |
| `agents_overview_actions.rs` | 对会话执行动作（归档 / 删除 / 重命名…） |

启动时它还会先确保 daemon 起来，失败时的提示很直白：

```text
The agents overview requires a shared server. Use codex --no-daemon to work without it.
```

典型场景：同时开着好几条会话（不同 worktree、不同任务）当调度台用；追父 agent 派生的
subagent；看到某条卡住了直接附着接管。

**放在一起看**：

| | `codex queue` | `codex agents` |
|---|---|---|
| 方向 | 向**已有会话**投递 | **浏览 / 选择**会话 |
| 界面 | 无，一条命令 | TUI 面板 |
| 是否产生 turn | ✅ 触发一次新 turn | ❌ 除非你附着或操作 |
| 对状态的要求 | 那条 thread **已加载** | **看到**所有 thread |
| 为什么必须共享 server | 队列要投给持有它的那个实例 | 要看到别的客户端开的会话 |

底层原因是同一个：**thread 状态归 app-server 持有**。所以任何"从外部操作一个 thread"的
功能，都必须连上持有它的那个 server。

#### 4. exec-server

`codex-rs/exec-server` 是一个独立的小 JSON-RPC 服务，通过 `codex-utils-pty`
生成和控制子进程。它存在的意义是**执行环境可以和 app-server 不在同一台机器上**：

```text
ws://IP:PORT                                     # 默认
--remote URL --environment-id ID [--name NAME]   # 注册为一个 environment
forward --connect ws://HOST:PORT --remote URL    # 转发
```

在 core 侧对应 `EnvironmentManager` 与 `TurnEnvironmentSelection`
（`SessionServices.turn_environments`）。

#### 5. code-mode host

Code Mode 的 V8 **跑在独立进程里**：

```rust
// code-mode/src/remote_session/connection.rs:158
let mut command = Command::new(host_program);
// :169
let mut child = command.spawn()...
```

`host_program` 来自 `InstallContext::code_mode_host_program()`，
即随包分发的 `codex-code-mode-host` 可执行文件（Windows 上是 `.exe`）。
主进程通过 gRPC 与它通信（`code-mode/src/grpc_session/`）。

隔离的理由是 V8 的崩溃、内存占用和 GC 都不该拖垮主进程。

#### 6. MCP servers

最常见形态是**子进程**：`rmcp-client/src/stdio_server_launcher.rs` 拉起 stdio server，
主进程通过标准输入输出说 MCP 协议。也支持 HTTP / WebSocket 形态。

#### 一次交互式会话实际起了哪些进程

这取决于启动参数，两种情况差别很大：

```text
[默认：无 --no-daemon，且未被其它条件排除]

codex（主进程，TUI）
  │  unix socket
  └──> app-server daemon（独立进程）
         ├── codex-linux-sandbox  子进程（按需）
         ├── apply_patch          子进程（按需）
         ├── MCP server(s)        子进程（按需，stdio）
         ├── 用户命令             子进程（shell / exec）
         └── code-mode host       子进程（仅启用 Code Mode 时）


[--no-daemon（或 --oss / --profile / 自定义 -c 等被排除的情况）]

codex（主进程，TUI）
  ├── app-server          同进程（Embedded）——不是子进程
  ├── codex-linux-sandbox 子进程（按需）
  ├── apply_patch         子进程（按需）
  ├── MCP server(s)       子进程（按需，stdio）
  ├── 用户命令            子进程（shell / exec）
  └── code-mode host      子进程（仅启用 Code Mode 时）
```

**特别注意**：上面两种都不是"客户端 + 服务端"的教科书结构。
单机交互式启动时，app-server 要么是同一进程里的一个模块，
要么是一个本地 daemon，**都不会走网络**（daemon 用的是 unix socket）。
真正跨机器的是 `--remote` 那条路径。

#### 三种部署拓扑

```text
[1] 默认：本地共享 daemon
    codex(TUI)  ─┐
    IDE 扩展      ├─ unix socket ──> app-server daemon ──> Session(s) ──> shell
    codex agents ┘

[2] --no-daemon 或被排除：单进程
    codex(TUI) ── 同进程 ──> app-server ──> Session ──> 本地 shell

[3] 远端工作区
    codex(TUI) ── ws ──> app-server(远端) ──> Session ──> exec-server ──> shell
```

### 基础结构：先认识这些名字

下面这些名字会在后续每一节反复出现。先给一张速查表，再逐个说清楚职责。

| 名字 | 一句话 | 位置 |
|---|---|---|
| `ThreadId` | 线程的持久句柄（UUIDv7） | `protocol/src/thread_id.rs` |
| `SessionId` | 历史遗留的会话 ID，值等于**根线程的 ID** | `protocol/src/session_id.rs` |
| `RolloutId` | 就是 `ThreadId`，用于 rollout 文件名 | `protocol/src/thread_id.rs:25` |
| `ThreadManager` | 门面：对外提供 start/resume/fork | `core/src/thread_manager.rs:238` |
| `ThreadManagerState` | 实现：活线程注册表 + 共享依赖 | `core/src/thread_manager.rs:402` |
| `StartThreadOptions` | **对外**启动参数（意图） | `core/src/thread_manager.rs:255` |
| `ThreadSpawnRequest` | **对内**信封（意图 + 继承的依赖） | `core/src/thread_manager.rs:318` |
| `NewThread` | 返回值三元组 | `core/src/thread_manager.rs:172` |
| `CodexThread` | 对外契约对象（`pub`） | `core/src/codex_thread.rs:201` |
| `Session` | 内存运行时（`pub(crate)`） | `core/src/session/session.rs:60` |
| `SessionState` | Session 内的可变状态 | `core/src/state/session.rs:69` |
| `SessionServices` | Session 内的只读依赖容器 | `core/src/state/service.rs:48` |
| `SessionIo` | 双向通道 + 终止信号 | `core/src/session/mod.rs:405` |
| `SessionSource` | 这条会话是**谁**开的 | `protocol/src/protocol.rs:2826` |
| `ThreadSource` | 这条线程在 Codex 内部**干什么** | `protocol/src/protocol.rs:2843` |
| `ThreadStartupMetadata` | 启动时的配置快照（无重放历史） | `core/src/thread_startup_metadata.rs:20` |
| `InitialHistory` | 启动时的历史来源（四态） | `history/src/lib.rs:366` |

#### `ThreadId` / `SessionId` / `RolloutId`

```rust
/// Identifier for a Codex thread.
///
/// Codex-generated thread IDs are UUIDv7, and some use cases rely on that.
pub struct ThreadId { pub(crate) uuid: Uuid }
```

三个要点：

- **都是 newtype**，不是裸 `String`。序列化成字符串、JSON Schema 也是 string，
  但 Rust 侧拿不到"其实是别的字符串"这种错误。
- **都用 UUIDv7**（`Uuid::now_v7()`）。`core/src/session/mod.rs` 里 `new_submission_id`
  的注释把原因写得很直白：*"some use cases take advantage of the fact that these are UUID7
  which encodes a timestamp, so think carefully before changing this."* ——时间有序，天然可排序。
- **`SessionId` 是历史遗留**。`SessionMeta` 里的注释：
  *"session_id is equal to the root thread's ID."*
  所以它不是第二个身份，而是同一条线程在旧版本里的名字。`SessionConfiguredEvent`
  的反序列化里专门做了回退（`protocol/src/protocol.rs:4075`）：

  ```rust
  thread_id: wire.thread_id.unwrap_or_else(|| wire.session_id.into()),
  ```

  今天它还有两处实际用途：agent control 的 `identity()`，以及 message board 的树键。

`RolloutId` 就是 `ThreadId` 的别名，但注释说明了它们的区别：
普通 rollout 文件用线程 ID 当 rollout ID；`thread/revert` 会**新建一个 rollout 文件、
用新的 rollout ID，但保留原线程 ID**。

#### `ThreadManager` vs `ThreadManagerState`

```rust
pub struct ThreadManager {
    state: Arc<ThreadManagerState>,
    _test_codex_home_guard: Option<TempCodexHomeGuard>,
}
```

`ThreadManager` 只有一个真实字段，是个**薄门面**：
对外提供 `start_thread` / `resume_*` / `fork_*` / `spawn_legacy_subagent`，
内部全部转发给 `state`。真正干活的是 `ThreadManagerState`（`impl` 从 `:1606` 开始），
`spawn_thread` 就在它身上（`:2040`）。

为什么拆两层？因为 `Arc<ThreadManagerState>` 要能被 clone 到各处
（比如扩展拿 `Weak<ThreadManager>` 去 spawn 子 agent），而门面本身保持轻量。

`ThreadManagerState` 的字段分成三类：

```rust
pub(crate) struct ThreadManagerState {
    // ① 活线程注册表
    threads: Arc<RwLock<HashMap<ThreadId, Arc<CodexThread>>>>,
    thread_created_tx: broadcast::Sender<ThreadId>,
    thread_id_generator: ThreadIdGenerator,

    // ② 跨线程共享的依赖
    auth_manager, models_manager, environment_manager,
    skills_service, plugins_manager, mcp_manager,
    code_mode_session_provider, extensions,
    user_instructions_provider, image_store, thread_store,
    agent_graph_store, attestation_provider, external_time_provider,

    // ③ 进程级身份
    session_source,        // 默认来源，每个 thread 可覆盖
    installation_id,
    analytics_events_client,
    ops_log,               // 测试用
}
```

`ThreadManager::new`（`thread_manager.rs:521`）的参数列表本身就是一份
"这个进程需要哪些依赖"的清单。注意里面的一个条件分支（`:557`）：

```rust
let code_mode_session_provider: Arc<dyn CodeModeSessionProvider> =
    if config.features.enabled(Feature::CodeModeHost)
        || config.code_mode.disable_in_process_fallback
    {
        Arc::new(ProcessOwnedCodeModeSessionProvider::default())   // 独立进程
    } else {
        Arc::new(DisabledCodeModeSessionProvider)
    };
```

——依赖是**在边界上按配置注入**的，core 不自己 new 全局单例。

#### 三个 envelope：`StartThreadOptions` / `ThreadSpawnRequest` / `NewThread`

这三个结构对应启动流程的三个方向，是理解"对外/对内分离"最好的例子：

```text
调用者 ──StartThreadOptions(pub)──> ThreadManager ──ThreadSpawnRequest(private)──> State
                                                                                    │
调用者 <──────────── NewThread { thread_id, thread, session_configured } ───────────┘
```

`ThreadSpawnRequest`（`:318`）比 `StartThreadOptions` 多了这些**继承来的运行时依赖**：

```rust
auth_manager, agent_control,
parent_thread_id, parent_originator, forked_from_thread_id,
fork_persistence, inherited_environments, inherited_instructions,
inherited_exec_policy, user_shell_override,
```

也就是：外部的调用者只表达"我要开一条什么样的线程"，而继承关系（父线程是谁、
认证和预算从哪继承）在内部补齐。这个模式在 Codex 里反复出现。

#### `CodexThread`：对外契约对象

```rust
pub struct CodexThread {
    pub(crate) session: Arc<Session>,
    pub(crate) io: SessionIo,
    // Queued agent mail owns a read guard until handled or dropped; eviction needs a write guard.
    pub(crate) residency_gate: Arc<RwLock<()>>,
    pub(crate) session_source: SessionSource,
    startup_metadata: ThreadStartupMetadata,
    rollout_path: Option<PathBuf>,
    out_of_band_elicitations: Mutex<OutOfBandElicitations>,
    _diagnostics_guard: GaugeGuard,
}
```

它是 **app-server 唯一直接操作的对象**。方法面很宽：

```text
submit / start_or_steer_turn / start_turn_if_idle / continue_turn_if_idle
steer_turn / suspend_turn_and_shutdown / recover_turn_if_idle
update_thread_settings / preview_thread_settings_overrides / checkpoint_thread_settings
flush_rollout / ensure_rollout_materialized / checkpoint_preparation
shutdown_and_wait / wait_until_terminated
emit_thread_idle_lifecycle_if_idle / emit_thread_ready_lifecycle
```

两个不显眼但重要的字段：

- `residency_gate` —— 注释说明了它解决什么：排队中的 agent mail 持有读锁，
  而淘汰（eviction）需要写锁。这是"排队消息不能被中途清掉"的保证。
- `_diagnostics_guard: GaugeGuard` —— 一个 drop 时自动更新活跃线程数的守卫，
  靠 RAII 而不是手工加减。

#### `Session` / `SessionState` / `SessionServices` / `SessionIo`

`Session` 是**内存运行时**（`pub(crate)`，外部看不到），职责见第一幕第 1 节。
它内部拆成三块，这个拆分本身就是一次重构的结果：

**`SessionState`（`core/src/state/session.rs:69`）**
文档注释是 *"Persistent, session-scoped state **previously stored directly on** `Session`."*
——原来这些字段直接挂在 `Session` 上，后来抽出来用一把 Mutex 保护。字段按用途分：

```text
配置     session_configuration, base_instructions_provenance
历史     history: ContextManager, history_reset: CancellationToken
用量     latest_rate_limits, latest_token_usage_record, auto_compact_window
回合衔接 previous_turn_settings, last_started_turn_id, next_turn_is_first
预热     startup_prewarm, shell_snapshot_prewarm
权限     granted_permissions_by_environment_id
其它     mcp_dependency_prompted, additional_context, current_time_reminder,
         active_connector_selection, pending_session_start_sources, shutting_down
```

注释里对 `last_started_turn_id` 的解释值得一读：
*"Latest task admitted in this runtime, retained across completion and history edits.
Cleared by standalone settings changes to invalidate pending continuation."*
——这是"续跑"能不能续的判据。

**`SessionServices`（`core/src/state/service.rs:48`）** —— 只读依赖容器，一次性注入。

**`SessionIo`（`core/src/session/mod.rs:405`）** —— 双向通道 + 终止信号，见第一幕第 1 节。

三者的分工可以概括成一句话：**Services 是不可变的依赖，State 是可变的会话状态，
Io 是与外界的边界。**

#### `SessionSource` vs `ThreadSource`

这两个最容易混。区别在于回答的问题不同：

| | 回答的问题 | 取值 |
|---|---|---|
| `SessionSource` | 这条会话是**谁**开的？ | `Cli` / `VSCode`（默认）/ `Exec` / `Mcp` / `Custom(String)` / `Internal(..)` / `SubAgent(..)` / `Unknown` |
| `ThreadSource` | 这条线程在内部**干什么**？ | `User` / `Subagent` / `GuardianReview` / `Feature(String)` / `MemoryConsolidation` |

`SessionSource` 的三个用途值得注意：

- 分析埋点（哪类客户端在用）
- `restriction_product()`（`ThreadManager::new` 里用它构造 skills/plugins 的受限视图）
- Guardian 的隔离判定（`crate::guardian::is_basic_session_source`）

它的子变体携带完整身份信息：

```rust
pub enum SubAgentSource {
    Review,
    Compact,
    ThreadSpawn {
        parent_thread_id: ThreadId,
        depth: i32,
        agent_path: Option<AgentPath>,      // 如 "/root"
        agent_nickname: Option<String>,
        agent_role: Option<String>,         // 别名 agent_type
    },
    MemoryConsolidation,
    Other(String),
}
```

注意 `InternalSessionSource` 只有两个值：`MemoryConsolidation` 和 `Guardian`——
说明只有这两类内部会话被当作"内部"处理。

#### `ThreadStartupMetadata`

```rust
/// The configuration reported when a thread started, without its replay history.
/// Current settings are available through [`CodexThread::config_snapshot`].
pub struct ThreadStartupMetadata {
    pub session_id: SessionId,
    pub thread_id: ThreadId,
    forked_from_id, parent_thread_id, thread_source, thread_name,
    model, model_provider_id, service_tier,
    approval_policy, approvals_reviewer,
    permission_profile, active_permission_profile,
    cwd, reasoning_effort, network_proxy, rollout_path,
}
```

它是 `SessionConfiguredEvent` **去掉 `initial_messages`** 之后的版本，两者可以互转：

```rust
impl From<&SessionConfiguredEvent> for ThreadStartupMetadata { ... }
fn to_session_configured_event(&self, initial_messages: Option<Vec<EventMsg>>) -> ...
```

为什么要留这一份？因为 `SessionConfiguredEvent` 里带 `initial_messages`
（可能是很长的重放历史，resume 时尤其大），而启动元数据要长期驻留在内存里。
拆开之后，**报告给客户端的内容**和**长期保留的配置**可以各管各的。

注释里还有一句提示：想看**当前**设置要调 `CodexThread::config_snapshot`，
因为这个结构记录的是**启动那一刻**的设置。

### 术语澄清：三种 "telemetry"

后面几节会反复出现 `x-codex-turn-metadata`、`SessionTelemetry`、`otel` 这些词。
它们**不是同一件事**，而且只有第三种是"厂商采集数据"。先分清楚，否则容易把功能字段当成遥测。

| # | 叫什么 | 出网吗 | 用途 |
|---|---|---|---|
| 1 | 本地遥测上下文 | ❌ | 给本地 OTEL 打标签用的请求上下文 |
| 2 | `x-codex-turn-metadata` 等请求头 | ✅ 发给模型服务 | **功能字段为主**，属于协议契约 |
| 3 | OTEL / analytics | ✅ 发给厂商 | 产品分析 |

**第 1 种：本地遥测上下文。**
`core/src/client.rs:2556` 的 `AuthRequestTelemetryContext` 就是典型：
记录"这次请求用哪种认证、带没带 header、是不是重试"。它不出网。
`client.rs` 开头那句 "reasoning controls, telemetry context, and turn metadata" 说的就是它。

**第 2 种：发给服务端的请求元数据。**
`core/src/responses_metadata.rs` 定义了几十个 key，通过 `x-codex-turn-metadata` 发出，
内容分两类：

```text
功能性（删了会坏功能）
  thread_id / turn_id / parent_thread_id / root_turn_id / forked_from_thread_id
  window_id / window_number / context_window_id     # 自动压缩窗口
  turn_started_at_unix_ms / tool_namespaces_info
  sandbox / sandbox_mode / subagent_kind / thread_source / agent_name

属性与归因
  installation_id / analytics_enabled / mcp_attribution / workspaces
  model / codex_version / reasoning_effort
```

它是**接口契约**而不是可选遥测，两个证据：`RESERVED_METADATA_KEYS` 禁止 app-server 客户端
覆盖 core 拥有的字段；`BACKWARD_COMPATIBLE_RESERVED_METADATA_KEYS` 要求某些 key
即使逻辑上删了也要保留名字，否则旧客户端会出问题。

**第 3 种：厂商遥测。**
`codex-rs/otel/` + `codex-rs/analytics/`。默认导出目标写在 `otel/src/config.rs`
（`https://ab.chatgpt.com/otlp/v1/metrics`），但 debug 构建下默认关闭。

开关在 `core/src/otel_init.rs:70`：

```rust
let metrics_exporter = if config.analytics_enabled.unwrap_or(default_analytics_enabled) {
    to_otel_exporter(&config.otel.metrics_exporter)
} else {
    OtelExporter::None
};
```

app-server 默认关闭 analytics，IDE 那类第一方客户端才通过
`--analytics-default-enabled` 打开。内容边界上，默认策略是"结构化指标 + 截断的工具结果"：

| 配置项 | 默认 |
|---|---|
| `otel.tool_result.max_bytes` | 2048（超出截断） |
| `otel.log_user_prompt` | false |
| `otel.log_agent_responses` | false |
| `otel.log_guardian_assessments` | false |

`ToolResultLogConfig` 的文档注释带一句直白警告：
*"This does not affect model-visible output. Raising it can expose more tool data to logs."*

### 设计原则与前置知识

#### 从这些结构里能读出来的八条约定

1. **对外/对内分离。** `StartThreadOptions`(pub) vs `ThreadSpawnRequest`(private)、
   `CodexThread`(pub) vs `Session`(pub(crate))、`ThreadManager`(门面) vs `ThreadManagerState`(实现)。
   每一对都是"外部只表达意图，内部补齐继承和实现细节"。
2. **依赖在边界注入。** `ThreadManager::new` 收 15 个参数不是因为设计差，
   而是因为 core 不做全局单例。好处是测试可以换 `ThreadStore`、换 `TimeProvider`、
   换 attestation。
3. **`Arc` 共享 + `Weak` 回引。** 扩展拿的是 `Weak<ThreadManager>`（见 `ext/agent`），
   避免循环引用导致线程永不释放。
4. **类型化 ID。** `ThreadId` / `SessionId` 都是 newtype，同时实现
   `Serialize/Deserialize/JsonSchema/TS` ——同一套类型要喂给 Rust、JSON 和 TypeScript SDK。
5. **UUIDv7。** 时间有序，可当排序键。`new_submission_id` 的注释明确警告别轻易改。
6. **向后兼容的序列化。** `SessionConfiguredEvent` 手写 `Wire` 结构 +
   `thread_id` 缺失时回退到 `session_id`，还接受旧的 `sandbox_policy` 字段并投影成
   `permission_profile`（`protocol/src/protocol.rs:4025` 起，`sandbox_policy` 字段在 `:4050`）。
   这是"旧 rollout 必须还能读"的代价。
7. **重构方式是搬，不是堆。** `SessionState` 的 "previously stored directly on `Session`"
   说明做法是把状态挪进独立结构，而不是让 `Session` 长成 god object。
8. **RAII 代替手工计数。** `_diagnostics_guard: GaugeGuard` 这类字段靠 drop 自动更新指标；
   下划线前缀表示"不读，只为生命周期存在"。

#### 读代码需要的前置知识

**Rust 异步与并发原语**

| 原语 | 在这份代码里的用途 |
|---|---|
| `tokio` 多线程 runtime | 主运行时；入口专门起线程跑 `block_on`（见第一幕第 1 节第 0 步） |
| `async_channel` bounded / unbounded | 提交通道有背压（512），事件流无界 |
| `tokio::sync::watch` | 读取最新状态而不阻塞（`agent_status`） |
| `tokio::sync::broadcast` | 一对多通知（`thread_created_tx`） |
| `tokio::sync::{Mutex, RwLock}` | 异步锁；`threads` 用 `RwLock` 允许多读 |
| `CancellationToken`（tokio-util） | 取消绑定到已丢弃历史的在途工作 |
| `futures::future::Shared` | `session_loop_termination`，让多个调用者等同一个关闭过程 |

**为什么到处是 `Box::pin`**

这不是风格偏好，是**避免大 future 压栈**。代码里至少四处注释解释了这件事：

```text
arg0/src/lib.rs:236     "Runtime::block_on otherwise runs the top-level future on the caller's OS stack."
cli/src/main.rs:1018    "Keep the CLI dispatcher off the runtime's stack while the TUI rebuilds a thread."
tui/src/lib.rs:2031     "Keep the large event-loop future out of the enclosing startup futures..."
core/src/session/mod.rs:511   #[inline(never)]
```

配套的一招是 `#[inline(never)]`（见 `Session::spawn`）：

```rust
/// Hide the concrete startup future from callers while keeping initialization lazy.
#[inline(never)]
pub(crate) fn spawn(args: SessionSpawnArgs) -> BoxFuture<'static, ...>
```

**feature flag 体系**

所有开关集中在 `codex-rs/features/src/lib.rs` 的 `Feature` 枚举，每个 spec 带
`Stage::{Stable, Experimental, UnderDevelopment, Deprecated, Removed}` 和 `default_enabled`。
运行时判定一律是 `config.features.enabled(Feature::X)`。
注意 `Stage::Removed` 的项会保留为 no-op，这是"配置向后兼容"的一部分。

**workspace 分层**

```text
protocol  纯数据类型（ThreadId / SessionSource / EventMsg…），不含运行时
   ↑
core      运行时（Session / tools / context / extensions）
   ↑
app-server 接口层（JSON-RPC v1/v2）
   ↑
tui / cli 前端
```

依赖只向上，不向下。`core` 不认识 `tui`，`tui` 也不能摸 `Session` 内部。

---

## 下一幕

准备好之后进入正文：

→ [第一幕：最小闭环](act-01-minimal-loop.md)
