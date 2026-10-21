# 第三幕：记忆与扩展 —— 让能力可增长

> **幕目标**：讲清"持久化 → 扩展框架 → 四类具体能力"这条链路。
> 这一幕是 Codex 架构最有特色的部分：用一个编译期插槽框架，
> 让 memory / skill / MCP / plugin 都不必长进 core。
>
> **前置依赖**：第一幕（上下文、world state）与第二幕（工具抽象、审批）。
>
> **状态**：第 9–13 节正文完成

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

第二幕结束时，一次工具调用已经有了完整结局：获准并执行、被拒绝、或者整轮中断。
但用户关掉进程以后，这些结论去了哪里？再次打开 Codex 时，为什么还能看到原来的对话、
工具结果、权限设置和任务状态？

这里先澄清三个容易混在一起的对象：

| 对象 | 本质 | 生命周期 |
|---|---|---|
| `Session` | 执行 turn 的运行时 | 一次进程内的 thread 生命周期 |
| thread history | 发给模型的消息序列 | 跨 turn、跨进程 |
| rollout | 追加写入的规范事件日志 | 跨进程，是恢复的真相源 |

恢复不是“把上次的 `Session` 从内存里找回来”。原来的内存对象已经没了。Codex 做的是：
读取持久化记录，重建一份模型历史与运行状态，再创建一个**新的 `Session`**接管同一个
`ThreadId`。因此本节讨论的不是简单反序列化，而是一次受约束的历史重放。

先看一个具体问题：上次停在一个工具调用执行到一半时，恢复后应该怎么办？
Codex 的答案不是猜测工具是否完成，而是依赖落盘顺序。模型发出调用时，
“请求执行这个 `call_id`”的记录已经先写入历史与 rollout；若进程随后退出，
恢复后至少能识别出这条调用意图，而不会把半条记录静默丢掉。工具结果是否已经存在，
则由后续记录决定。这个顺序在第二幕 §7 已经出现过，现在把它放回完整持久化链路里看。

### ② 代码入口

先走“运行中写什么”，再走“重新启动读什么”。两条链最后都在
`Session::record_initial_history`（`codex-rs/core/src/session/mod.rs:1536`）汇合。

#### 场景 A：turn 运行中，记录怎样增量落盘

接点在第二幕 §7 的工具调用和 §8 的审批结果：

```text
core/src/session/turn.rs                模型 item / turn event 产生
  └─ core/src/session/mod.rs:4465       Session::persist_rollout_items
       └─ thread-store/src/live_thread.rs:238
            LiveThread::append_items
            └─ thread-store/src/local/live_writer.rs:317
                 write_and_project
                 └─ :362  durable_write
                      ├─ RolloutRecorder::record_canonical_items
                      └─ RolloutRecorder::flush
                 └─ materialize_to_sqlite        # 尽力而为的可查询镜像
```

`persist_rollout_items` 是一个很窄的闸口：core 只把已经归一化的 `RolloutItem` 交给
`LiveThread`。至于本地文件、远端 thread store 还是别的实现，由 `ThreadStore` 抽象决定。
`--no-daemon` 主线使用 `LocalThreadStore`，所以下一站是 JSONL `RolloutRecorder`。

每次 append 都会做两件事：

1. `record_canonical_items` 把记录追加成 JSONL；
2. `flush` 形成持久化屏障。

然后，如果当前是较新的 history mode，`LocalThreadStore` 才尝试把同一批数据投影到 SQLite。
错误只记录告警，不反向阻止已经成功的 JSONL 写入。这条先后顺序说明了两者的地位：

> JSONL rollout 是规范历史；SQLite 是可重建的查询视图。

这里也解释了为什么关闭进程不是第一次“保存”。工具调用、审批、world state、token usage
都在发生时就写。`Session::shutdown_and_wait`
（`codex-rs/core/src/session/mod.rs:1049`）只提交 `Op::Shutdown` 并等待 session loop 退出；
它负责收尾，不负责把整段会话在退出时一次性序列化。

#### 场景 B：执行 `codex resume`

app-server 先根据 thread id 或 rollout path 找到存储记录：

```text
app-server/src/request_processors/thread_processor.rs:4578
  load_resume_initial_history_from_stored_thread
  ├─ Paginated
  │    └─ :4582  ThreadStore::load_latest_model_context
  │         └─ InitialHistory::Resumed(ResumedHistory { history, rollout_path, .. })
  └─ Legacy
       └─ read_stored_thread_for_resume
            └─ :4678  stored_thread_to_initial_history
```

两种 history mode 的读取入口不同，但最终都构造 `InitialHistory::Resumed`。
随后 `ThreadManager` 创建新 thread，`Session::record_initial_history` 按四种初始历史分流：

```text
InitialHistory::New
  → 没有旧历史；第一次真实 turn 前才安装初始上下文

InitialHistory::Cleared
  → 也没有旧历史；但保留“这是 clear 启动”的语义，供 hooks 等消费者区分

InitialHistory::Resumed
  → 重放同一 thread 的 rollout，并恢复 token usage / settings / world state

InitialHistory::Forked(items)
  → 把源 thread 的已选快照重建成子 thread 的初始历史
  → 子 thread 的 ThreadId 由本次 fork 单独分配，不存在于 items 里
```

恢复分支的核心不是 `history` 字段赋值，而是：

```text
record_initial_history
  └─ apply_rollout_reconstruction        core/src/session/mod.rs:1703
       └─ reconstruct_history_from_rollout
            core/src/session/rollout_reconstruction.rs:169
       ├─ 恢复模型历史
       ├─ 恢复 world state baseline
       ├─ 恢复 reference TurnContext
       ├─ 恢复 previous turn settings
       ├─ 恢复 retained context / Guardian checkpoint
       └─ 恢复 context window 编号与 id
```

thread 注册完成后，`ThreadManager` 还会调用
`CodexThread::emit_thread_resume_lifecycle`
（`codex-rs/core/src/thread_manager.rs:2366`）。扩展在这里重新水合自己的
`thread_store`，例如恢复 goal、queue 或 memory 相关状态。这是“core 历史已经恢复”
之后的第二层恢复，详见 §10。

### ③ 关键设计

#### a. `RolloutItem` 不是“聊天消息列表”

`RolloutItem`（`codex-rs/history/src/lib.rs:201`）是规范日志的记录类型，包含：

| 变体 | 记录什么 | 恢复时是否进入模型历史 |
|---|---|---|
| `SessionMeta` | thread id、cwd、来源、初始配置、fork 关系 | 不直接进入；提供元数据 |
| `ResponseItem` | 用户/助手消息、function/custom tool call 及其输出 | 是 |
| `InterAgentCommunication` | 多智能体通信 | 转成模型输入后进入 |
| `TurnContext` | 某轮冻结的模型、权限、上下文基准 | 不直接进入；恢复运行时设置 |
| `WorldState` | 全量快照或 JSON merge patch | 不直接进入；重放成 world state baseline |
| `Compacted` | 压缩摘要与 replacement history | 作为新的历史检查点 |
| `TokenUsageRecord` | token 统计检查点 | 不进入；恢复 UI / 预算状态 |
| `RetainedContext` | 压缩后仍需要保留的结构化上下文 | 部分恢复进上下文管理器 |
| `SecurityRiskScore` | Guardian 等异步安全评分 | 不进入 |
| `RealtimeItem` | 实时呈现所需的稀疏记录 | 不进入普通模型历史 |
| `EventMsg` | TurnStarted/Complete/Aborted、settings、rollback 等运行事件 | 多数不直接进入，但决定重放边界 |

所以“resume 恢复会话”实际恢复了至少五类东西：模型消息、turn 边界、运行配置、
world state 和扩展私有状态。只恢复第一条会让模型看起来接着聊，却丢掉动作权限和状态；
只恢复配置则没有对话上下文。

#### b. 增量落盘：规范日志先于查询镜像

`RolloutLine`（`codex-rs/history/src/lib.rs:350`）把每条 item 连同 `timestamp`、可选
`ordinal` 编成一行 JSON。选择追加式 JSONL 的直接收益是：

- 崩溃时最多丢失当前未 flush 的尾部，不需要重写整个会话文件；
- 新 item 可以从尾部追加，天然符合第一幕“不重写历史”的约束；
- 出问题时仍然能删除坏行或截断尾部，而不是修一个二进制数据库页。

但纯 JSONL 不擅长列表、搜索、标题和元数据查询，于是 SQLite 投影存在。
`codex-thread-store/README.md` 把方向写得很明确：raw history append 只追加历史；
thread metadata 走单独的 `update_thread_metadata` API；本地存储再用 JSONL 作为规范来源、
SQLite 作为可查询镜像。

这也是为什么 `write_and_project` 的顺序不能反过来：

```text
durable_write(JSONL)
  → materialize_to_sqlite
```

如果 SQLite 先写成功而 JSONL 失败，查询层会出现一个无法由真相源重建的“幽灵状态”；
反过来，SQLite 暂时落后只是索引旧了，可以下次重建。

#### c. 恢复历史是从尾部反向重放

rollout 是追加日志，但恢复时不能天真地把所有 `ResponseItem` 顺序塞回去。原因来自第二幕之后
新增的几类边界：

- compaction 之后，旧历史可能已经是死分支；
- thread rollback 会删除最近 N 个用户 turn；
- `TurnContext` 决定某个 turn 使用的设置；
- `ThreadRolledBack`、`TurnStarted`、`TurnComplete` 等事件定义 turn 的边界；
- world state 是 full snapshot 加 patch，不是普通文本。

`reconstruct_history_from_rollout`
（`codex-rs/core/src/session/rollout_reconstruction.rs:169`）因此从尾部向前扫。
反向扫描的意义是：先遇到最新状态，遇到旧的检查点时再决定哪些更早记录已经不需要重放。

第一步选“最新且可用的 compaction”：

```text
select_input_compaction
  ├─ 找最后一个 Compacted
  ├─ 必须带 replacement_history
  ├─ window/version 条件必须满足
  └─ 返回 compaction + suffix
```

若最新 compaction 本身不完整，不能退而使用更旧的 compaction 来“凑一个历史”。
那样会把新检查点之后可能发生过的状态改回旧值。

第二步反向分组为 turn segment。大致规则是：

```text
看到 ResponseItem / UserMessage / TurnContext / WorldState
  → 放进当前 segment

看到 TurnStarted
  → 当前 segment 有明确左边界，finalize

看到 ThreadRolledBack(n)
  → 记下要跳过的 n 个包含真实用户输入的 segment

看到 Compacted
  → 让更早的上下文基准失效，并把 replacement_history 当作候选历史基座
```

顺序反过来做，`ThreadRolledBack(1)` 只需跳过“从尾部数第 1 个用户 turn”，
不必先把所有记录展开再删除。

第三步构造最终模型历史。若存在 replacement history，先安装它，再顺序重放它之后的 suffix：

- `ResponseItem` 走 `replay_annotated_item`；
- `InterAgentCommunication` 转成模型输入；
- `ThreadRolledBack` 调 `drop_last_n_user_turns`；
- 其它事件只影响运行状态，不直接成为消息。

因此恢复后的模型历史可能和 rollout 文件长度完全不同。rollout 保留审计痕迹，
重建结果只保留“在最新检查点下仍然有效的模型可见历史”。

#### d. world state 与运行设置也重放

第一幕 §5 已经说明，world state 在 rollout 中有两种记录：

- `full = true` 的全量 snapshot；
- 之后的 RFC 7386 merge patch。

恢复时先按时间顺序翻回正向，再执行：

```text
Compacted
  → world_state_baseline = None

WorldState(full)
  → baseline = snapshot

WorldState(patch)
  → baseline.apply_merge_patch(patch)
```

孤立 patch 会被忽略并告警，因为没有完整基准可以应用。这个规则和第一幕 §5 的
模型侧差异渲染共用同一套 `WorldStateSnapshot`，所以恢复后的下一次采样能继续只发送变化。

除了 world state，重放还择取最新的：

- `PreviousTurnSettings`：上轮使用的模型、comp hash 等；
- `reference_context_item`：重建初始上下文时用于比较的 TurnContext；
- `last_started_turn_id`：最后启动但可能未完成的 turn；
- `retained_context` 与 `guardian_history`：压缩后仍需保留的审查 / 指令状态；
- context window 的编号和 id。

这些都不是“聊天文本”，却会直接影响下一轮能否正确继续，所以它们和模型历史一起从同一份
rollout 重建。

#### e. resume、clear、fork 是三种不同操作

把三种操作的差异压缩成表：

| 操作 | 初始历史 | ThreadId | 旧历史 | 典型用途 |
|---|---|---|---|---|
| resume | `Resumed(ResumedHistory)` | **保持不变** | 重放 | 继续原来的 thread |
| clear | `Cleared` | **新分配** | 不带入 | 清空上下文后重新开始 |
| fork | `Forked(items)` | **新分配** | 选择性带入 | 从某个历史点开新分支 |

先消除一个最容易产生的误解：`InitialHistory::Forked(items)` **不包含“子 thread 应该是哪个
ThreadId”这个字段**。`items` 只是从源 thread 选出的 rollout 快照；子 thread 的身份是另一个
输入参数。用 A 表示源 thread，B 表示这次 fork 创建的 thread：

```text
源 thread A
  │
  │ thread/fork(A)
  ├─ 读取并裁剪 A 的历史 → items
  ├─ 为本次 fork 预留新 ThreadId B
  └─ InitialHistory::Forked(items) + reserved_thread_id = B
       │
       └─ Session::spawn
            ├─ 当前 Session.thread_id = B
            ├─ SessionMeta.id = B
            ├─ SessionMeta.forked_from_id = Some(A)
            ├─ 从 items 重建 B 的模型历史 / world state
            └─ 把选定的继承内容写入 B 的 rollout

结果：A 和 B 两个 thread。
不存在“某个 fork 请求自己的第三个 ThreadId”。
```

所以“在新 ThreadId 下建立 thread”指的是：**相对源 thread A 新建 B**。B 不是另一层中间
对象，它正是 `thread/fork` 最终返回、后续可以继续对话和 resume 的那个 thread。
如果之后从 B 再 fork，才会分配另一个 C；fork 操作不会修改 A 或 B 的既有 id。

代码里可以把身份与历史分开看：

```text
app-server/src/request_processors/thread_processor.rs:5161
  reserved_thread_id = 为 fork 预留的新 id B

app-server/src/request_processors/thread_processor.rs:5181
  fork_options.reserved_thread_id = Some(B)
  history = InitialHistory::Forked(items_from_A)

core/src/thread_manager.rs:1468
  fork_thread_from_history(...)
    ├─ history 只负责快照内容
    ├─ forked_from_thread_id = Some(A)
    └─ spawn_thread(options)
         └─ options.reserved_thread_id = Some(B)

core/src/session/session.rs:853
  match (initial_history, reserved_thread_id)
    Forked(_) + Some(B)  → thread_id = B
    Resumed(A) + None    → thread_id = A

core/src/session/session.rs:1017
  CreateThreadParams {
    thread_id: B,
    session_id: B,                 # root session；不是 A
    forked_from_id: Some(A),
  }
  → 新建 B 自己的 rollout / SessionMeta

core/src/session/mod.rs:1622
  InitialHistory::Forked(items)
    → apply_rollout_reconstruction(items)
    → 得到 B 启动时的模型历史和运行状态
```

paginated source 的 fork 还有一层容易误读的中间表示：
`fork_prepared_thread`（`codex-rs/core/src/thread_manager.rs:1489`）一开始先把 prepared
context 包成 `InitialHistory::Resumed(A)`，因为 thread store 的 prepare API 要用 A 的身份
描述“从哪份历史准备快照”。但 `fork_thread_with_initial_history` 会在 spawn 前：

```text
从 Resumed(A) 取出 source_thread_id = A
  → forked_from_thread_id = Some(A)
  → fork_history_from_snapshot 把 Resumed(A) 变成 Forked(items)
  → 仍把 app-server 预留的 B 作为 reserved_thread_id
```

所以这里的 `Resumed(A)` 是准备 fork 时的内部来源描述，并没有启动 A、恢复 A 或复用 A 的
ThreadId；真正进入 `Session::spawn` 的仍是带 B 的 `Forked(items)`。

这里有两个不同选择：

```text
Resumed(A) + reserved_thread_id = None
  → 复用 A，resume 原 thread

Forked(items) + reserved_thread_id = Some(B)
  → 新建 B，把 A 的快照装进 B
```

`Session::spawn` 甚至明确拒绝 `Resumed(_) + Some(新 id)` 的组合，因为“恢复原 thread”和
“给恢复历史换一个新身份”是两种不同语义。fork 必须先把源历史转换成 `Forked(items)`，
再由 B 创建自己的 `Session`。

接着看 `record_initial_history` 的 `Forked` 分支到底做了什么：

1. 给快照中缺失的 response item 补稳定 id；
2. 用 B 的配置创建默认 `TurnContext`；
3. 对 `items` 调 `apply_rollout_reconstruction`，按 §9 ③c 的规则重建模型历史；
4. 从快照恢复 token usage、world state baseline 和上轮设置；
5. 追加 B 自己当前的 thread settings；
6. 根据 `ForkPersistence` 写 B 的 rollout。

第 2、3 步使用的是 **B 的当前配置**，不是照抄 A 的 live process。A 的旧 `TurnContext`
只作为快照里的历史证据参与重建；fork 之后模型、权限等仍由 B 的有效配置决定。
源快照里的 `SessionMeta(A)` 可以随复制历史保留作审计/血缘信息，但运行时身份由创建
B 时传入的 `thread_id=B` 决定，模型可见历史也不会把 `SessionMeta` 当成聊天消息。

`ForkPersistence` 决定怎样保存 B 的 rollout：

- `Copied`：把选中的 A 快照复制到 B 自己的 rollout；
- `Referenced`：B 的 rollout 记录 history base / 继承范围，只持久化本地新增部分；
- 两者都先在内存里重建同一份 B 历史，区别只在磁盘表示。

因此“重放父 thread 快照”更准确的说法是：**把源 thread 已冻结的 `items` 重建成新 thread
B 的初始内存状态**。它不是重新 resume A，也不会打开或继续 A 的 `Session`；A 保持原样。

`/clear` 不是给现有 `Session` 发一条“请忘记历史”的消息。app-server 的
`ThreadStartSource::Clear` 在 `thread_processor.rs:1504` 被映射成
`InitialHistory::Cleared`，随后通过 `thread/start` 创建新 thread。旧 thread 仍然在存储里，
新 thread 只是不加载它的历史。这样“清空”不会破坏可恢复性，也不需要改写旧日志。

fork 则先读取源 thread，再把快照包成 `InitialHistory::Forked`。`SessionMeta.forked_from_id`
保留来源关系；它让 UI 能表达 A → B 的血缘，也让存储层判断历史复制方式和 fork 边界。
它和 `parent_thread_id` 也不是同一个概念：前者描述“从哪条 thread fork 出来”，后者描述
agent / subagent 的树形归属。

#### f. SQLite 镜像为什么存在，但不拥有真相

这件事容易读错：`state` crate 既是 thread list 的数据库，也参与 memory job、goal 等状态，
看起来像“所有历史的数据库”。但 thread history 的主关系仍然是：

```text
rollout JSONL  → 规范历史
SQLite         → 可查询投影 / 运行元数据
```

SQLite 让以下操作不必每次扫描整个目录和 JSONL：

- 列出 thread、按时间排序和分页；
- 搜索标题、预览和元数据；
- 找到某 thread 当前的 rollout path；
- 读取 paginated history 的最新模型上下文；
- 保存 memory pipeline 的 claim、watermark 和阶段输出。

即使投影损坏或暂时不可用，本地 rollout 仍能作为恢复来源。反过来说，**只备份 SQLite 不够**，
只备份 JSONL 也需要重建索引。两者不是互相替代，而是“规范记录 + 可重建索引”。

#### g. 冷 rollout 压缩是另一种生命周期

长时间使用后，旧 rollout 很多，逐个读取成本上升。`codex-rollout` 支持把冷文件压缩成
`.jsonl.zst`，读取器对 `.jsonl` 和 `.jsonl.zst` 透明处理；需要继续追加时再物化回 plain JSONL。

这与 §14 的“上下文压缩”不是一回事：

- 这里的压缩是**存储编码压缩**，解压后仍是原来的 JSONL；
- §14 的 compaction 是**语义压缩**，会生成 replacement history，丢弃不再发送给模型的旧消息。

两者都要求保留兼容边界：读取器要能识别 plain/compressed 切换，恢复逻辑要能读懂旧版
`CompactedItem`。存储优化不能悄悄改变历史含义。

### ④ 取舍

**为什么不用 SQLite 直接保存全部会话？**

因为主历史的需求首先是 append-only 和可审计，不是关系查询。JSONL 的写入模型简单，
崩溃后的恢复边界清楚，也能直接查看一条记录。SQLite 更适合索引和事务，把两者分开后，
查询层可以随时重建，历史层仍然容易追踪。

代价是必须维护投影一致性和恢复时的去重。项目用 JSONL 先写、SQLite 后投影的顺序控制风险，
并在 shutdown 时再做一次 materialize，但它本质上仍是两套表示。

**为什么恢复要反向扫描，而不是正向过滤？**

正向扫描在遇到 rollback 时很别扭：你已经把旧 segment 放进了历史，却要在后面某条事件到来时
删除它；遇到最新 compaction 时也无法提前跳过更老记录。反向扫描先把“最新有效边界”找出来，
再按 turn 边界 finalize，更接近“从当前状态往回找存活分支”的语义。

代价是实现复杂。turn id 兼容、未完成 turn、legacy compaction、world-state patch 顺序
都必须在这个循环里处理，任何新 rollout item 都要明确它是否参与分段。

**为什么 resume 不复用原 `Session` 对象？**

因为内存中的 `Session` 绑定通道、取消 token、MCP 连接、环境句柄和后台任务。
进程重启后这些对象都无法恢复，只恢复其输入状态才是可靠做法。新 `Session` 重新建立服务，
再用相同 `ThreadId` 接管历史和数据目录。

代价是“恢复”必须同时重建多个层次；少恢复一个 extension store，核心历史可能正确，
但 goal、queue 或 schema 等能力会表现成新 thread。

**为什么保留 clear 与 new 两个空历史状态？**

模型可见内容都为空，但运行语义不同：clear 有“用户主动清除”的上下文，session-start hooks
会收到对应的 `SessionStartSource::Clear`；new 则是正常新建。把两者合并会让日志和 hook
无法区分用户意图。

回到主流程：一次 resume 完成后，控制流回到普通 `turn/start` 入口。模型历史、world state
和 token 状态已经装回新的 `Session`；下一节要回答的是，这些恢复动作为什么能由
extension 在不改 core 的情况下补上。

---

## 10. Contributor 框架

### ① 问题

§9 已经出现了两种“core 之外”的需求：

- resume 时，goal / queue 等能力要在 core 重建完 thread 后恢复自己的数据；
- 每个 turn 开始前，memory / skill / MCP 要决定给模型增加哪些上下文和工具。

如果每一项都在 core 里写一个 `if feature_enabled { ... }`，core 会同时承担流程编排、
具体能力语义和外部依赖。更糟的是，每安装一个能力都要修改主循环，违反第二幕形成的执行链边界。

Contributor 框架把这件事反过来：

> core 定义稳定的生命周期插槽，扩展把自己的能力注册进去；core 只调用 trait，
> 不依赖具体扩展 crate。

这也是项目规则里“尽量别往 `codex-core` 加新概念”的直接动机。新增能力应优先成为独立
extension crate，而不是继续膨胀主 crate。

### ② 代码入口

主装配点只有一个：

```text
app-server/src/extensions.rs:50
  thread_extensions(dependencies)
    ├─ queue / goal / history-notes / git / guardian / web-search / image-gen
    ├─ memories
    ├─ mcp
    ├─ plugins
    └─ skills
         → ExtensionRegistryBuilder::build()
         → Arc<ExtensionRegistry<Config>>
```

`thread_extensions` 接受 app-server 已经构造好的依赖（认证、state DB、analytics、
environment manager、tool provider 等），把每个 extension 需要的依赖注入构造函数，
最终产出一个不可变 registry。

registry 被传给 `ThreadManager`，每个新 `Session` 都持有同一个 `Arc`。trait 对象和它们
捕获的进程级依赖共享；每个 thread / turn 的可变状态则放进 `ExtensionData`：

```text
thread 创建
  ├─ thread-store/src/live_thread.rs      建立持久化句柄
  ├─ core/src/session/session.rs:995      thread_extension_data = new_with_init(...)
  └─ session.rs:1651                      依次调用 on_thread_start

thread 注册完成
  └─ core/src/thread_manager.rs:2361      on_thread_ready

resume
  └─ core/src/thread_manager.rs:2366      on_thread_resume
```

三者不是同义重复：

- `on_thread_start`：session 基础对象已经建好，但 thread 还未注册到 host；
- `on_thread_ready`：thread 已经可被查找、订阅和中断；
- `on_thread_resume`：历史已从 rollout 重建，扩展可以恢复自己的持久状态。

一个 turn 的主调用顺序可以按执行时间概括为：

```text
turn/start
  ├─ TurnStartAdmission::admit_turn_start           # 可选准入门
  ├─ TurnStarted event
  ├─ TurnLifecycle(BeforeTaskRegistration)
  ├─ task 注册，用户可中断
  ├─ TurnLifecycle(RegularTaskStart)
  │    └─ 需要时先刷新 MCP runtime
  ├─ TurnInputContributor                           # 记录/注入本轮用户输入
  ├─ capture StepContext
  │    ├─ ToolContributor::tools_for_step
  │    └─ McpRuntime::current_binding
  ├─ ContextContributor::contribute_thread/turn_context
  ├─ ContextContributor::contribute_world_state
  ├─ ModelRequestContributor                        # 请求发出前
  ├─ 模型流
  │    ├─ TurnItemContributor / TurnLifecycle::on_item_completed
  │    └─ ToolLifecycleContributor
  ├─ ApprovalReviewContributor                      # 仅在审批点调用
  ├─ TokenUsageContributor
  └─ TurnLifecycle::on_turn_stop / abort / error

thread 空闲
  └─ ThreadLifecycleContributor::on_thread_idle

thread 关闭
  └─ ThreadLifecycleContributor::on_thread_stop
```

不是每个 turn 都必然走完全部节点；例如只做 warmup 的请求没有普通 `RegularTaskStart`，
无工具调用时也不会进入 tool lifecycle。重点在于：**扩展只能在这些明确时机参与，
不能拥有或重写主循环。**

### ③ 关键设计

#### a. 13 个 trait 是 13 个稳定接点

`ext/extension-api/src/contributors.rs` 和 `model_request.rs` 一共定义 13 个 contributor trait。
它们按“什么时候需要扩展”划分，而不是按功能名字划分：

| Trait | 核心方法 | 被调用的时机 | 典型用途 |
|---|---|---|---|
| `ThreadLifecycleContributor` | start / ready / resume / idle / stop | thread 生命周期节点 | goal、queue、状态水合 |
| `TurnLifecycleContributor` | start / item / stop / abort / error | turn 生命周期节点 | 计数、清理、失败处理 |
| `TurnInputContributor` | `contribute` | 收到一次新 turn 输入后 | 注入显式 skill、用户相关上下文 |
| `ContextContributor` | thread / turn / world state | prompt 组装 | prompt cache 友好的上下文与工具状态 |
| `ToolContributor` | `tools` / `tools_for_step` | 每个采样 step 建工具表 | memory / skills 的原生工具 |
| `ToolLifecycleContributor` | dispatch / start / finish / timing | 工具执行前后 | telemetry、审计、MCP 结果观察 |
| `ApprovalReviewContributor` | `decide` | 工具请求审批 | Guardian 或自动审查 |
| `McpServerContributor` | `contribute` / `selected_plugins` | MCP runtime 刷新 | 把插件、Apps、选择根变成服务器配置 |
| `SkillInvocationContributor` | `on_skill_invocation` | 显式或隐式 skill 被使用时 | skill 使用指标与实验 |
| `TokenUsageContributor` | `on_token_usage` | 每次记录模型 token usage | goal 预算、预算 UI 状态 |
| `ConfigContributor` | `on_config_changed` | thread 配置成功提交后 | 扩展缓存配置快照 |
| `TurnItemContributor` | `contribute(&mut TurnItem)` | 一个完整 item 被解析后、发送前 | 后处理和附加 metadata |
| `ModelRequestContributor` | `request` → interceptor | 模型请求发出前 | 按请求包装响应流 |

这 13 个接口约束了扩展能触达的边界。例如扩展可以贡献工具，但不能替换 `run_turn`；
可以逐个修改 `TurnItem`，但不能改变模型请求的历史前缀规则。

#### b. registry 是“有序广播”，少数位置是“认领”

构建完成后 registry 不再添加 contributor，内部是十三组 `Vec<Arc<dyn Trait>>`。
大多数方法按注册顺序遍历：

```text
for contributor in registry.context_contributors() {
    fragments.extend(contributor.contribute_*(...).await);
}
```

所以不同扩展对同一类上下文的贡献是**叠加**关系，顺序稳定但不做隐式覆盖。
工具表出现重名时，由 `ToolRegistry` 的冲突和优先级规则处理，而不是 contributor
按“最后写入赢”静默覆盖 core。

审批是不同语义：

```text
registry.decide_approval(input)
  → 按注册顺序询问
  → 第一个返回 Some(decision) 的 contributor 认领
  → 后面的 reviewer 不再执行
```

这是“认领”，因为一个审批请求只能有一个最终决定。`TurnStartAdmission` 也是准入门：
它返回一个 permit 或拒绝，核心不能把多个扩展的准入结果简单相加。

`ExtensionEventSink` 则是单向出口，不是 contributor。扩展产生的 queue update、goal update、
warning 通过 sink 回到 app-server，再由 host 转成对应的 notification；扩展不直接持有
WebSocket 连接或 RPC 广播逻辑。

#### c. 三层 `ExtensionData` 解决状态归属

`ExtensionData`（`codex-rs/ext/extension-api/src/state.rs:49`）不是通用的全局 map，
而是按 host 对象挂载的类型化附件：

```rust
struct ExtensionData {
    level_id: String,
    entries: Mutex<HashMap<TypeId, Arc<dyn Any + Send + Sync>>>,
}
```

三个作用域分别是：

| Store | 所有者 | 生命周期 | 适合放什么 |
|---|---|---|---|
| `session_store` | `SessionServices` | 本次 `Session` | 环境、auth、MCP resource client 等 session 依赖 |
| `thread_store` | thread runtime | thread（resume 时重建） | 配置快照、goal、最近 skill、插件选择、缓存 |
| `turn_store` | `TurnContext` | 单个 turn | 本轮 catalog、选中 skill、warning、临时决策 |

`turn_store` 在某些工具回调里被命名为 `step_store`，因为工具表按采样 step 捕获；
但它仍在 turn 生命周期内，不构成第四个跨 turn 层级。

这些 store 的核心契约不是“方便共享”，而是“不要把持久状态藏在 extension 对象里”。
extension value 本身跨 thread 共享，因此不能持有某个用户的当前会话状态；会话相关状态必须
放进对应 store。`ThreadResumeInput` 之所以有意义，就是因为 `thread_store` 是新的，
扩展要主动从持久数据重新填充。

`get_or_init` 和 `insert_if` 在锁内完成检查和插入，避免多个并发调用看到旧值后一起初始化。
这是类型擦除容器为异步 runtime 提供的最小并发保证。

#### d. `thread_extensions()` 决定“装什么”，不决定“何时调用”

装配函数把具体能力接入 registry：

- `codex_queue_extension`、`codex_goal_extension` 管理跨 turn 的用户队列与目标；
- `codex_history_notes_extension` 生成会话历史提示；
- `codex_git_attribution`、`codex_web_search_extension`、
  `codex_image_generation_extension` 提供各自工具或上下文；
- `codex_guardian_v2` 安装审批 reviewer 和风险评分；
- `codex_memories_extension` 安装 memory 读写；
- `codex_mcp_extension::install` 安装 hosted Apps server；
- `install_plugins` 让插件选择参与 MCP server contribution；
- `codex_skills_extension` 同时注册 thread、turn、context、input、tool 和 invocation 接点。

因此一个会话拥有哪些能力由 app-server 的装配函数决定；调用先后由 core 的生命周期决定。
扩展作者只关心“我的能力属于哪个接点”，不需要知道 `run_turn` 内部有多少行。

#### e. core 的消费方式：只拿接口需要的数据

看两个实际消费点：

```text
context:
core/src/session/world_state.rs:296
  for contributor in context_contributors():
      contributor.contribute_world_state(input)

tools:
core/src/tools/spec_plan.rs:337
  for contributor in tool_contributors():
      contributor.tools_for_step(session_store, thread_store, step_store)
```

输入结构里只暴露稳定 id、配置快照、环境、previous world state 和三个 store。
core 不会把整个可变的 `Session` 交给扩展。扩展拿不到主动修改 `run_turn` 局部变量的通道，
只能返回声明式结果，由 core 决定怎样合并、记录和持久化。

这就是依赖倒置的具体形态：

```text
app-server（知道全部具体能力）
  → ExtensionRegistry（trait 集合）
    → core（只按生命周期调用 trait）
```

#### f. 生命周期相位避免“太早”和“太晚”

`TurnStartPhase` 只有两个值：

- `BeforeTaskRegistration`：确定 turn 合法后、任务注册前；
- `RegularTaskStart`：任务已注册，可被中断，普通任务开始准备。

需要读取和冻结 turn 起始 token 的 contributor 必须在前一相；需要 MCP runtime 的扩展
可以在后一相声明 `requires_mcp_runtime`，core 会在回调前刷新 MCP。
普通任务准备可以被取消，所以扩展还必须容忍“start 没跑完就收到 abort”。

这比提供一个笼统的 `on_turn()` 更啰嗦，但把可取消边界、MCP 可用性和 token snapshot
变成了显式契约，避免扩展靠执行到一半再猜当前系统处于什么阶段。

### ④ 取舍

**为什么是编译期 trait，不是运行期脚本插件？**

编译期 trait 让 core 能获得静态类型、取消传播、`Send + Sync` 约束和确定性注册顺序。
扩展可以深度接入 turn 生命周期，同时不把任意脚本的动态行为带进主循环。

代价是封闭性：新增接点要改 extension API 和 core；第三方不能只丢一个文件就接管
任意生命周期；扩展与 host 版本存在 ABI/API 兼容成本。Codex 接受了这个成本，
换来的是生命周期可审计。

**为什么不允许扩展改主循环？**

主循环承载了第一幕的上下文前缀、第二幕的并发与审批顺序，以及 thread/session 的取消语义。
一旦扩展可以任意插入或重排 step，这些全系统不变量就无法局部证明。当前设计只开放
“在某一步补充什么”或“观察某一步发生了什么”，更像受控的 middleware contract，
不是开放的执行图。

代价是某些需求需要由 core 先增加一个通用接点，而不能由扩展自行绕路。

**contributor、plugin、hook 到底怎么区分？**

- **contributor** 是 host 内部 Rust trait，作用在 thread / turn / context / tool 等固定接点；
- **plugin** 是声明式资源包，声明 skills、MCP servers、apps 和 hooks；
- **hook** 是某类外部事件处理器，由 hook runtime 调度，可调用命令或 MCP tool。

plugin 本身不实现 contributor；它是更上层的分发载体。例如 plugin 里的 skill 最终进入
skills extension 的 catalog，plugin 里的 MCP server 最终由 MCP extension 贡献给 runtime。
三层关系会在 §13 再闭合。

**为什么 registry 构建后必须不可变？**

如果在运行中随手安装 contributor，已经发出的 prompt、正在执行的工具和进行中的 turn
会看到不同的扩展集合，恢复语义也会变得不可定义。不可变 registry 保证同一 thread 内的
能力集合稳定；变化被收敛到配置快照、world state 或 MCP runtime 的显式发布点。

回到主流程：`thread_extensions()` 装配出的能力并不会自动进入模型上下文。接下来从
最典型的“只增加知识与流程、不启动外部 server”的能力开始，看 Skill 如何经过 catalog、
选择、资源读取和 world state，最终变成模型看见的说明。

---

## 11. Skill

### ① 问题

§10 已经装好了 `codex-skills-extension`，但“安装能力”不等于“模型知道怎么使用它”。
假设用户输入：

> 用 `$imagegen` 生成一张产品示意图，尺寸要求看它自己的说明。

这里同时出现三个问题：

1. 模型怎么知道存在一个叫 `imagegen` 的 skill；
2. 它怎么找到正确的 `SKILL.md`，又怎么按需读取 `references/` 或运行 `scripts/`；
3. 这些文件来自本机、remote executor、云包还是插件，读取权限为什么不会串线。

Skill 的答案不是“把某个 Python 函数注册成工具”，而是把一份 Markdown 指令包变成
模型可见的能力目录；模型先用目录找到它，再通过普通读取或 `skills.read` 获取内容。
真正执行仍走第二幕的 shell / MCP / 其它工具。

### ② 代码入口

Skill 从“随二进制发布”到“进入某轮 prompt”分成四段。

#### 第一段：内置 skill 写入磁盘

内置样本在编译时进入二进制：

```text
codex-rs/skills/src/lib.rs:55
  include_dir!("$CARGO_MANIFEST_DIR/src/assets/samples")
  └─ :77  install_system_skills(codex_home)
       ├─ 计算嵌入目录指纹
       ├─ 已匹配 CODEX_HOME/skills/.system/.codex-system-skills.marker → 跳过
       └─ 否则重建 .system 目录并写 marker
```

`SKILL.md` 的 frontmatter 由 `codex_skills::parse_skill_frontmatter_metadata`
（`codex-rs/skills/src/parser.rs:44`）解析。必需字段是 `description`；`name` 缺失时可由目录名
补出；可选的 `metadata.short-description` 用于更短的目录展示。

#### 第二段：按来源发现 catalog

app-server 装配 skills extension 时提供三类 provider：

```text
app-server/src/extensions.rs:107
  SkillProviders
    ├─ HostSkillProvider
    ├─ ExecutorSkillProvider
    └─ （可选）Cloud provider
```

发现结果不是裸文件列表，而是 `SkillCatalog`：

```text
SkillCatalog
  entries: Vec<SkillCatalogEntry>
  warnings: Vec<String>

SkillCatalogEntry
  ├─ id: SkillPackageId
  ├─ authority: SkillAuthority { kind, id }
  ├─ main_prompt: SkillResourceId
  ├─ 展示 name / description / short_description
  ├─ display_path / alias_root / dependencies
  └─ enabled / prompt_visible
```

`SkillAuthority` 是关键。一个 catalog entry 不仅告诉模型“叫什么”，还告诉 Codex
“该由谁读取这个包”。`SkillSourceKind` 有：

- `Host`：本机 host 文件系统，包括 bundled、用户、仓库和已安装插件 skill；
- `Executor`：remote executor 自己的文件系统；
- `Cloud`：云侧包，资源通过 cloud provider 读取；
- `Custom`：留给其它 provider 的扩展点。

因此不能把 `skill://...` 简单当成本地路径，也不能拿 host 的文件句柄去读 executor skill。
provider 的 `list` / `read` / `search` 都以 authority 和 package id 路由。

#### 第三段：每轮把 catalog 放到模型能看到的位置

SkillsExtension 同时实现了 `ContextContributor` 和 `TurnInputContributor`：

```text
core/src/session/mod.rs:4198
  build_initial_context_with_world_state
  └─ ContextContributor::contribute_thread_context

core/src/session/world_state.rs:296
  build_world_state_for_step
  └─ ContextContributor::contribute_world_state

core/src/session/turn.rs:1184
  build_extension_turn_input_items
  └─ TurnInputContributor::contribute
```

三种入口分工如下：

- thread context：稳定的、启动时即可知道的目录和规则；
- world state：环境相关、可能随 step 变化的 executor / cloud / host catalog；
- turn input：本轮用户输入中显式提到的 skill，以及对应主 prompt。

`ext/skills/src/extension.rs:185` 的 thread catalog 和
`world_state_catalogs.rs` 的 step catalog 都会按预算渲染，不是无限拼接。

#### 第四段：识别显式调用并读取主文件

`TurnInputContributor::contribute`（`codex-rs/ext/skills/src/extension.rs:375`）按顺序做：

```text
1. 汇总 executor step / host snapshot / cloud snapshot 的 catalog
2. collect_explicit_skill_mentions(user_input, catalog)
3. 可选：运行 shadow selection 实验
4. 对选中 entry 调 read_main_prompt
5. 把内容包装成 SkillInstructions fragment
6. 截断超长主 prompt 并发出 warning
7. 把 catalog / selected / warning 状态放进 turn_store
```

显式调用有两个入口：

- 结构化 `UserInput::Skill { name, path, .. }`；
- 文本里的 `$skill-name`，或 `[$skill-name](skill://.../SKILL.md)` 形式的带路径 mention。

`collect_explicit_skill_mentions`（`codex-rs/skills/src/selection.rs:42`）先按精确 path 匹配，
再按名字匹配；同名 skill 不唯一时不能靠“猜一个”，只接受无歧义的名字。

读到主文件后，扩展可能把它作为本轮 user-context fragment 注入；更长内容或 executor/cloud
资源则由 `skills.list` / `skills.read` 工具分页读取。`skills.read` 的 handle 上限是 2,048 字节，
单次响应预算 512 KiB，并支持 cursor 分页（`codex-rs/ext/skills/src/tools/mod.rs:56`）。

### ③ 关键设计

#### a. 一个 Skill 是“文档 + 可选资源”，不是函数

典型目录结构是：

```text
my-skill/
  SKILL.md
  references/
    details.md
  scripts/
    helper.py
  assets/
    diagram.png
```

`SKILL.md` 的 frontmatter 负责目录级元数据，正文负责触发条件、步骤和资源路由。
`references/`、`scripts/`、`assets/` 只有在正文要求时才读取或执行。

这与“把函数 schema 放进工具表”有本质区别：

- skill 改变模型的工作方法和上下文；
- tool 改变 host 能执行什么动作；
- skill 可以指导模型调用已有工具，却不自动获得新执行权限。

因此 skill 可以在不增加 host 可执行面的情况下扩展能力。代价是它依赖模型正确阅读和遵守说明，
比强类型函数调用更“软”。

#### b. 内置、host、executor、cloud 解决的是不同部署问题

四类来源对应四种文件所有权：

| 来源 | 文件在哪 | 读取方式 | 解决的问题 |
|---|---|---|---|
| bundled/system | Codex 二进制内，运行时写到 `CODEX_HOME/skills/.system` | host 文件系统 | 官方基础能力随版本交付 |
| host/user/repo | 本机 `CODEX_HOME` 或项目目录 | host provider | 用户自定义流程 |
| executor | remote 执行环境的文件系统 | exec-server filesystem / discovery | 技能和执行环境同机 |
| cloud | 云端包 | cloud/MCP resource provider | 不依赖本地文件分发 |

插件安装的 skill 也归入 host 来源，但额外带 `plugin_id` 和 plugin namespace，便于归因和
共享资源策略。executor 插件里的 skill 则保留 executor authority，不能因为 host 上也存在
同名目录就混用。

#### c. catalog 与正文分离控制上下文成本

如果每轮把所有 `SKILL.md` 全文注入，几个技能就会吃掉大量上下文。Codex 只常驻展示：

- `name`；
- `description` / 可选 short description；
- 来源 locator；
- 少量使用规则。

正文只在显式选择时读取，其它资源再由模型按需读取。技能目录的预算由
`skill_metadata_budget`（`codex-rs/ext/skills/src/render.rs:126`）计算：

- 配置了 `skill_max_context_tokens`：使用配置值，但最多 10,000 tokens；
- 否则按模型 context window 的约 2%；
- 没有 context window 时退化为约 8,000 字符。

主 prompt 另有 `MAX_SKILL_PROMPT_BYTES = 8_000` 的硬上限
（`render.rs:21`）。超限不是静默截断：`extension.rs:489` 会发出 warning，并把截断状态记入
本轮 skill 状态。

这套“目录常驻 + 正文按需 + 资源按需”的设计，和 §4、§5 的增量上下文原则一致：
不会为了偶尔用到的技能，在每一步都重复发送全文。

#### d. executor / cloud skill 通过工具跨权限边界

host skill 通常可以直接用 shell 打开 `SKILL.md`，因为模型和执行环境共享 host 文件系统。
remote executor 或 cloud 包不行：路径属于另一个 authority，普通 `exec_command` 未必能访问。

所以 skills extension 提供两个工具：

- `skills.list`：浏览当前 turn 可见的 executor / cloud package；
- `skills.read`：按 package + resource 读取 `SKILL.md` 或其中资源，支持分页。

工具读取时会把 authority 一起带上，再回调对应 provider。executor provider 还会拿到本 step
冻结的 executor roots 和 sandbox context；cloud provider 可能通过 MCP resource 读取。
这正是 §10 中 `ToolContributor::tools_for_step` 的用途：工具权限必须绑定到当前 step 的
环境快照，不能在另一个环境里复用。

#### e. 显式和隐式调用不是同一件事

显式调用是用户意图：

- `$deploy`；
- 带资源路径的 mention；
- UI 产生的结构化 `UserInput::Skill`。

这些输入会被 `TurnInputContributor` 选中，主动读取主 prompt 并注入。

隐式调用是模型在完成任务时，基于 catalog 描述自行决定读取某个 skill。此时：

1. 模型看到目录里的名称和描述；
2. 它通过 host 读取或 `skills.read` 获取正文；
3. `SkillInvocationContributor` 或命令观察逻辑识别出它确实读了这个 skill / script；
4. 使用事件进入 analytics 或影子实验。

`detect_implicit_skill_invocation`
（`codex-rs/ext/skills/src/invocation.rs:17`）会识别读取 skill 的 `SKILL.md` 或执行
其 `scripts/` 下资源的命令。它不强制模型“已经正确执行”，只记录文件访问这一可观察事实。

#### f. 动态选择器为什么只做影子

`CheapSkillSelector`（`codex-rs/ext/skills/src/dynamic_skill_selector.rs:48`）定义一组
确定性、无副作用的选择算法。当前实验并行运行多种方法：

- weighted lexical；
- fielded BM25；
- character n-gram；
- multi-query lexical；
- RRF 融合 lexical / character；
- routing-card lexical；
- LRU 与 lexical / routing 的组合；
- 基于最近 turn 的 task-context fusion。

`ShadowSelectionExperiment`（`ext/skills/src/shadow_selection_experiment/mod.rs`）在每轮
用这些算法给候选 skill 排名，但**不修改模型可见 catalog，也不自动加载选中的 skill**。
后续真正发生的 skill invocation 才用于给各算法打 hit / rank 指标。

这样可以在不改变用户体验的前提下收集离线评估数据。若直接把某个 selector 接到 catalog，
一旦选错，模型会看不到正确技能；影子模式把“算法评估”和“线上能力”隔开。

影子实验临时存在，代码顶部也标明 evaluation 后应删除。这说明它是实验设施，不是
Skill 架构的永久组成部分。

#### g. 整个世界没有向量 RAG

对这个提交下的 Rust 源码全量搜索，`embedding` 的命中只出现在“embedding app / embedding
caller / embedding host”等英文语境，没有向量索引、向量数据库或语义检索实现。
Skill 选择采用的是词法、BM25、character n-gram、RRF、LRU 和 routing card。

五种实际可见的“检索替代方案”是：

1. catalog 常驻少量结构化元数据；
2. 显式 `$name` / path mention 直接选择；
3. host 文件系统读取；
4. BM25 / n-gram 等轻量词法排名；
5. `tool_search` 对 deferred tool 做 BM25 检索。

这不是“Codex 不会语义理解”，而是检索层先选择低成本、可解释、无需维护向量库的方案。
模型仍可根据自然语言描述决定读哪个 skill。

### ④ 取舍

**为什么 skill 是提示词包，而不是可执行插件？**

提示词包可以直接复用 shell、MCP、审批和 sandbox，不需要定义新的 ABI，也不需要让每个 skill
自带运行时。升级一个 Markdown 文件通常比升级原生扩展安全，且 host / executor / cloud
都能用同一种 catalog 模型表达。

代价是能力边界依赖模型遵守说明。skill 能建议执行 `scripts/deploy.py`，但最终仍要通过
第二幕的工具审批和沙箱。若作者期待“加载 skill 就自动执行”，会高估它的能力。

**为什么目录和正文分离？**

因为“可能用到的技能数量”远大于“本轮真正需要的技能数量”。目录让模型知道有什么，
正文按需读取；如果直接全文注入，上下文成本和 prompt cache 抖动都会失控。

代价是模型必须先做一次读取，增加一个 turn step；如果描述写得太差，模型也可能选不中。
这也是 skill metadata 要求单行、长度受限并做预算化渲染的原因。

**为什么没有向量检索？**

向量检索需要额外 embedding 模型、索引生命周期、增量更新和一致性问题。当前 skill catalog
规模可以由元数据 + 词法排序覆盖；显式 mention 又给了高精度快捷路径。先不引入向量库，
可以保持本地、可解释和低运维成本。

代价是纯语义近邻、同义改写和跨语言召回会弱于 embedding。代码没有把这种能力“否认”掉，
而是投入 shadow selection 比较不同词法算法，等数据证明收益后再决定是否正式启用。

**为什么影子选择器不能直接改变 catalog？**

选错 skill 比不选更危险：它可能把完全无关的流程注入模型，或者让真正需要的 skill 被预算挤掉。
影子模式让所有算法都看到相同输入、记录同一套结果，但保留原有 catalog 作为控制组。

代价是线上仍依赖模型自行选择，实验价值要等后续 invocation 数据才能体现。

回到主流程：Skill 解决的是“把知识和流程带进本轮”。下一节的 Memory 解决另一半问题：
知识不来自用户当场给的包，而来自 Codex 过去完成的任务；它需要在后台先被抽取、固化，
再在之后的会话中进入同一批 contributor 接点。

---

## 12. Memory

### ① 问题

Skill 由用户或仓库当场提供；Memory 面对的是另一种信息：

> 上周用户说过发布说明要按什么格式写；昨天排除了某个方案；这个仓库的测试命令有特殊约定。
> 这些事实没有出现在当前 prompt 里，也不应该要求每条新会话重新讲一遍。

于是要解决两个方向相反的问题：

- 读：新会话怎样在工作开始前拿到高信号的跨会话记忆；
- 写：一次任务结束后，怎样从 rollout 中抽取可复用事实，又不会把每条对话都直接塞进长期记忆。

Memory 不是“把历史向量化后全部召回”。它把写入拆成后台两阶段，把读取拆成
“常驻摘要 + 按需检索原始证据”，并明确要求模型区分历史偏好与当前事实。

### ② 代码入口

先看一次新 turn 怎样进入 memory 读路径，再看后台 pipeline 怎样生成这些文件。

#### 场景 A：新 turn 读取 memory

MemoriesExtension 在 thread start 时记录 effective config：

```text
app-server/src/extensions.rs:97
  codex_memories_extension::install
    ├─ ThreadLifecycleContributor::on_thread_start
    │    └─ thread_store.insert(MemoriesExtensionConfig)
    ├─ ContextContributor::contribute_thread_context
    │    └─ ext/memories/src/prompts.rs:35
    │         build_memory_tool_developer_instructions
    └─ ToolContributor::tools
         └─ ext/memories/src/tools/mod.rs:28
              memory_tools(...)
```

读路径的入口是 §10 的 thread context：

```text
core/src/session/mod.rs:4198
  build_initial_context_with_world_state
    └─ ContextContributor::contribute_thread_context
         └─ MemoriesExtension
              ├─ 读 CODEX_HOME/memories{_v2}/memory_summary.md
              ├─ 按 2,500 token 上限截断摘要
              ├─ 渲染 read_path 模板
              └─ 返回 developer policy fragment
```

V1 把整段渲染结果作为一个 fragment；V2 再按每片最多 8,900 字节切开
（`ext/memories/src/extension.rs:82`），避免单个 `ContextualUserFragment` 超过系统上限。
这不是两套完全不同的 prompt，而是同一 read-path 内容为了 fragment 合约做的分片。

模型还可以在需要时调用 `memories` namespace 下的四个工具：

```text
memories.add_ad_hoc_note
memories.list
memories.read
memories.search
```

它们由 `ToolContributor` 注册，只有 `config.enabled && config.dedicated_tools` 时可见。
`search` 是文件内容子串检索，支持多 query、同 line / line window、context lines、
case sensitivity 和 cursor；它不是向量检索。

#### 场景 B：一次用户输入启动后台写 pipeline

app-server 在一次 turn 真正开始后触发：

```text
app-server/src/request_processors/turn_processor.rs:689
  start_memories_startup_task(...)
    └─ memories/write/src/start.rs:24
         ├─ 过滤 ephemeral / feature disabled / subagent
         ├─ 每个 MemoryVersion 准备一个 config
         ├─ ensure_layout(memory root)
         ├─ seed_extension_instructions
         ├─ phase1::prune
         ├─ rate-limit guard
         ├─ phase1::run
         └─ phase2::run
```

它是 `tokio::spawn` 的后台任务，不阻塞当前 turn。`dual_write` 打开时，V1 和 V2 各跑一条
pipeline；否则只跑配置的版本。

Phase 1 的入口是 `memories/write/src/phase1.rs:56`：

```text
claim_startup_jobs
  → buffer_unordered(8)
       → 读取一个 rollout
       → V1: 过滤并序列化 ResponseItem
       → V2: tiered input，最多占模型有效上下文的 70%
       → 严格 JSON output schema
       → secret redaction
       → 写 state DB stage-1 output
  → aggregate metrics
```

Phase 2 的入口是 `memories/write/src/phase2.rs:49`：

```text
claim global phase-2 job
  → prepare git baseline
  → build locked-down consolidation agent config
  → load bounded stage-1 selection
  → sync raw_memories.md / rollout_summaries/
  → prune stale extension resources
  → memory_workspace_diff
  → no changes? validate + succeed
  → write phase2_workspace_diff.md
  → spawn consolidation agent
  → validate artifacts + reset baseline + mark success
```

### ③ 关键设计

#### a. Memory 与 thread history 是两张不同账本

thread history 记录“这次任务完整发生了什么”，按 thread 隔离，恢复时逐项重放。
Memory 记录“过去哪些信息值得在未来任务中复用”，跨 thread 聚合，并通过文件结构重新组织。

两者的边界可以从目录看出。默认 V1 根目录是 `CODEX_HOME/memories`，V2 是
`CODEX_HOME/memories_v2`：

```text
memories/
  MEMORY.md
  memory_summary.md
  raw_memories.md
  rollout_summaries/
    <timestamp>-<hash>-<slug>.md
  extensions/
    ad_hoc/notes/
    <extension_name>/
      instructions.md
      resources/
  phase2_workspace_diff.md      # 只在 Phase 2 运行期间生成
```

不同文件回答不同问题：

| 文件 | 读者 | 作用 |
|---|---|---|
| `memory_summary.md` | 每个新会话的模型 | 最高信号、常驻摘要 |
| `MEMORY.md` | 按需读取的模型 | 更完整的主题索引 |
| `raw_memories.md` | consolidation agent | stage-1 结果的稳定合并视图 |
| `rollout_summaries/*.md` | 模型 / consolidation agent | 某条历史 thread 的证据 |
| `extensions/ad_hoc/notes/` | consolidation agent | 用户显式 remember / forget / correct |

因此记忆不是一段不断变长的对话，而是“摘要 + 索引 + 证据文件”的分层存储。

#### b. 读路径先给摘要，再给受控检索工具

`memory_summary.md` 会先经过 2,500 token 截断，再填入 read-path 模板。模板同时定义：

- memory 是历史上下文，不是当前行为的证明；
- 当前指令优先于 memory；
- 只有实际使用了 memory 才引用；
- 不引用 `memory_summary.md` 本身；
- 有需要时读取 matching rollout summary，而不是每轮猜测性检索；
- 只有用户明确要求 remember / forget / correction 时才写 ad hoc note。

这套规则处理的是 LLM memory 最危险的两个错误：

1. **时效错误**：把旧偏好当成当前仓库事实；
2. **无来源推断**：记得“某件事”却说不出来自哪条记录。

引用格式是回复末尾的 `<oai-mem-citation>` 块，列出相对 path、line range、note 和
rollout UUID。`codex-memories-read` 的 `citations.rs` 负责解析它，而不是让上层自己拆字符串。
提示词还明确不要在 pull request 中写这类引用。

#### c. `list` / `read` / `search` 是有界文件工具

read-path 工具直接面对 memory 目录，因此参数边界很具体：

| 工具 | 默认 / 上限 | 安全边界 |
|---|---|---|
| `memories.list` | 默认 2,000，最大 2,000 项 | 只列 relative path 下的直接文件/目录 |
| `memories.read` | 20,000 tokens | 只读 memory root 内文件，支持 line offset |
| `memories.search` | 默认 200，最大 200 matches | 子串 / 多 query / window，支持 cursor |
| `memories.add_ad_hoc_note` | 单文件 | 只 create，不覆盖 |

`LocalMemoriesBackend::resolve_scoped_path`（`ext/memories/src/local.rs:22`）拒绝：

- `..`；
- 绝对路径和 Windows prefix；
- hidden component；
- 路径中经过符号链接；
- 经过非目录节点继续向下。

search 遍历时也跳过 hidden path 和 symlink。这些检查不是“防模型恶意”这么简单；
它们保证 memory root 的边界不会被一个错误路径变量悄悄扩大到整个 home 目录。

#### d. ad-hoc note 是显式记忆意图的追加层

当用户说“记住这个约定”“忘掉上次那条”“更正一下”时，模型不应该直接编辑
`MEMORY.md` 或 `memory_summary.md`。系统给的合法写入路径是：

```text
memories.add_ad_hoc_note(
  filename = "2026-10-21T12-34-56-release-notes.md",
  note = "..."
)
```

本地实现（`ext/memories/src/local/ad_hoc_note.rs:17`）要求：

- 文件名匹配 `YYYY-MM-DDTHH-MM-SS-<slug>.md`；
- slug 只含小写 ASCII、数字和 hyphen；
- 使用 `create_new(true)`，同名文件直接失败，不覆盖；
- notes 目录逐级检查 symlink。

这是一层 append-only 意图日志。真正的去重、冲突处理和长期摘要更新留给 Phase 2；
因此一次“记住”不会立刻重写全局记忆，避免主 turn 和后台 consolidation 同时修改同一文件。

#### e. Phase 1 把单条 rollout 变成 stage-1 输出

Phase 1 不是按“用户说了什么”简单截取，而是对满足条件的旧 thread 跑一次抽取模型。
选择 job 时使用 state DB claim：

- 扫描最多 5,000 个 thread；
- 只考虑允许的交互 source；
- 限制 age、idle 时间和每次启动最大处理数；
- 用 lease 防止多个 startup 同时处理同一 thread；
- 失败写 retry delay，成功后保存 source updated time。

并发是 8，抽取模型默认 reasoning effort 为 `Low`。V1 输出三个字段：

```json
{
  "raw_memory": "...",
  "rollout_summary": "...",
  "rollout_slug": "release-notes"
}
```

V2 不再生成单独的 raw memory，只保留 summary + slug；summary 会被压到 9,000 bytes。
两版输出都经过 secret redaction 后才写 DB。

rollout 输入也要有界。V1 过滤掉非模型消息，只序列化 memory-relevant response item；
V2 使用 tiered input，并把输入限制在模型有效上下文的 70%。若无有效 context window 元数据，
fallback 是 150,000 tokens。

这一步的输出不是最终 `MEMORY.md`，而是 per-thread 的标准化证据。这样抽取可以并行，
失败可以独立重试，也不会让某个正在活跃的 thread 被半截总结。

#### f. Phase 2 用 workspace diff 驱动全局 consolidation

Phase 2 一次只允许一个全局 worker。它先读取有界 stage-1 集合，再把这些结果同步成：

- 每条 rollout summary 一个文件；
- V1 的合并 `raw_memories.md`；
- 保留最近的 extension resources。

memory root 内维护一个本地 git baseline。同步输入后，`memory_workspace_diff` 会生成从上次
成功基线到当前 worktree 的 git-style diff；写入 `phase2_workspace_diff.md` 时最多保留
4 MiB。若没有变化且 artifact 合法，Phase 2 直接成功，不花一次 consolidation 模型调用。

有变化时，后台启动一个专用 consolidation agent。正常 managed profile 下：

- cwd 是 memory root；
- 可写范围只有 memory root；
- network 关闭；
- approval policy 是 `Never`；
- `Collab`、`MemoryTool`、`Apps`、`Plugins` 被关闭；
- reasoning effort 是 `Medium`。

若父会话显式选择 `PermissionProfile::Disabled` 或 `External`，配置保留该选择，而不是
强行伪造 sandbox 能力。这是对“用户已经选择外部权限模型”的尊重；pipeline 仍然关闭
审批、网络相关能力与递归扩展。

agent 只能通过 memory 文件和 workspace diff 工作。完成后校验：

- V1 必须有 `MEMORY.md`；
- `memory_summary.md` 第一行必须是 `v1`；
- V2 summary 长度小于 10,000 bytes，且包含约定 headings；
- memory workspace 中不能留下 symlink。

校验通过后，只有在 worker 仍持有 lease 时才 reset git baseline；否则新的 worker 可能已经
接管，旧 worker 不应把新状态覆盖掉。

#### g. 读使用量与写成本都需要观测

read path 不只靠模型自称“我用了 memory”。`codex-memories-read` 的 `usage.rs` 会解析 shell
命令，把读取 `MEMORY.md`、`memory_summary.md`、`raw_memories.md`、rollout summary 或
skills 的行为归类，并附上 memory version。这样后续能评估哪些层真的被使用。

write path 则记录 token usage、job status、失败原因、e2e 时长和输出类型。
Phase 1 的 claim / lease / retry 让写入可以并发；Phase 2 的 global lock 和 heartbeat
让全局 consolidation 可以安全重试。

#### h. 三个 feature 开关控制不同面

代码中至少有三个相关能力开关，含义不同：

- `MemoryTool`：允许 memory extension 注入读上下文、注册 tools，并启动 pipeline；
- `ExternalAgentMemoryImport`：导入外部 agent 的 memory/config 数据；
- `Chronicle`：与记忆化历史相关的另一条 feature surface。

另外还有配置级开关 `memories.use_memories`、`memories.generate_memories`、
`dedicated_tools`、`dual_write` 和 version。feature gate 决定“能力是否存在”，配置决定
“存在后这一 thread 是否读、是否写、用哪些工具”。两者不能混为一谈。

### ④ 取舍

**为什么用 LLM 抽取，而不是每轮对全部历史做向量检索？**

这里的目标不只是“找出相似文本”，而是把一次任务压缩成可行动的事实、偏好和证据指针。
LLM 可以区分临时排错步骤与稳定项目约定，也可以把多条证据合并成简洁摘要；纯 top-k
向量检索只返回相似片段，仍要另一次模型整理。

代价是抽取和 consolidation 都要花模型调用，且可能产生幻觉。Codex 用两阶段职责分离、
严格 schema、secret redaction、workspace diff、artifact 校验和 citation 规则降低风险，
但没有把 memory 当成事实数据库。

**为什么拆成 Phase 1 / Phase 2？**

Phase 1 是按 thread 独立、可并行、可重试的归一化；Phase 2 是跨 thread 的全局冲突合并。
如果合成一个阶段，多个 worker 会同时重写同一份 `MEMORY.md`，同时还要承担逐 thread
失败重试，锁粒度会很差。

代价是数据要经过 state DB 中转一次，pipeline 更长；但每段都有明确输入、输出和失败恢复点。

**为什么用 git workspace diff 驱动 Phase 2？**

全局记忆是许多 Markdown 文件，consolidation 需要知道“这次输入变化了哪些内容、
删掉了哪些 extension resource”。git 已经能提供稳定的 add/modify/delete 语义和 unified diff，
比应用层再维护一套文件版本更直接。

代价是 memory root 多了一套 `.git` baseline；必须严格处理 baseline 重置、diff 文件自身排除、
symlink 和失败重试，否则 diff 会污染下一次输入。

**为什么 memory 不是“当前行为的证明”？**

用户偏好会变，项目会迁移，旧 rollout 中的路径和命令也可能过期。若模型把旧记忆当成当前
代码事实，它会忽略现场证据并输出看似有来源的错误结论。read-path 因此要求对会漂移、
有影响的结论在实际源码中验证，并诚实表达不确定性。

代价是 memory 不能直接消除所有重复查询；它主要帮助定位经验、约定和线索，而不是替代验证。

**为什么显式 remember 只追加 note？**

主 turn 直接编辑 consolidation 产物，会和后台 Phase 2 同时写同一批文件，也会绕过冲突处理。
追加一条带时间戳的 note 则保留用户意图和发生顺序，再由下一次 consolidation 决定怎样并入
摘要与索引。

代价是用户刚说“记住”后，立即读取全局摘要不一定马上看到；新信息要等后台合并完成。

回到主流程：到这里，持久化、扩展插槽、Skill 和 Memory 都已经落在同一个 thread/turn 生命周期里。
最后要处理的是“能力来自外部系统”：外部服务怎样被发现、延迟加载、审批、认证，
以及 plugin 和 Apps 怎样把一组外部资源装配成统一能力。

---

## 13. MCP + Plugin + Apps

### ① 问题

前三节扩展的能力仍然主要运行在 Codex 自己的进程和文件里。真正接入 GitHub、Slack、
Google Drive 或自建服务时，问题变成：

- 外部进程提供哪些工具，Codex 在什么时候启动它；
- 工具表太大时，怎样不让每个 schema 都常驻 prompt；
- 工具调用应该绑定到哪个 server、哪份 catalog、哪个认证账号；
- plugin 怎样打包 skills、MCP servers 和 apps，同时不变成任意代码注入；
- “用户点一下安装”怎样被模型请求、用户确认、后端安装和 runtime 刷新串起来。

本节沿一条真实调用链展开：

```text
用户说“查一下我 GitHub 上这个 PR”
  → plugin/app config 被投影成 MCP server
  → MCP runtime 启动或复用连接
  → step 捕获不可变 tool catalog
  → 模型直接调用或先 tool_search
  → MCP handler 绑定 prepared call
  → 走第二幕审批 / sandbox / 结果回灌
```

MCP、Plugin 和 Apps 不是三套互不相干的平行协议。它们最终都可以收敛成 MCP server
或声明式资源，再由 §10 的 contributor 框架接入主流程。

### ② 代码入口

#### 场景 A：配置和 plugin 怎样变成 MCP server

`thread_extensions()` 安装两类 MCP contributor：

```text
app-server/src/extensions.rs:98
  codex_mcp_extension::install
    → HostedPluginRuntimeExtension
         └─ 在 Feature::Apps 开启时贡献 codex_apps server

app-server/src/extensions.rs:99
  codex_mcp_extension::install_plugins
    → PluginContributor
         ├─ executor selected roots → plugin manifest / MCP / apps
         └─ cloud plugin catalog → hosted connector attribution
```

core 在 session / turn 配置变化时构造 `McpRuntimeInput`：

```text
core/src/session/mcp_runtime.rs:316
  build_mcp_runtime_input
    ├─ effective_mcp_servers(config, auth)
    ├─ 给每个 server 绑定本 turn 的 permission profile
    ├─ startup_policy:
    │    SubAgent  → LazyWhenCached
    │    其它      → Eager
    └─ connector / tool catalog cache / client extensions
         ↓
core/src/session/mcp_runtime.rs:283
  publish_mcp_runtime
    └─ McpRuntime::replace
         codex-mcp/src/runtime.rs:312
         └─ McpConnectionSet::new
              codex-mcp/src/connection_manager.rs:227
```

新的 runtime 通过 `ArcSwap` 发布；旧的 `McpBinding` 仍然持有它需要的连接和 catalog，
不会因为下一次刷新而在调用中途被替换。

#### 场景 B：每个 step 捕获工具

第一幕 §2 的 step capture 在这里展开：

```text
core/src/session/mod.rs:3720
  capture_step_context_inner
    ├─ McpRuntime::current_binding_with_requirements
    │    codex-mcp/src/runtime.rs:418
    └─ turn::built_tools
         └─ build_tool_router
              ├─ append_mcp_tools
              ├─ extension ToolContributor
              ├─ append_tool_search_executor
              └─ ToolRouter::model_visible_specs
```

`McpBinding` 是一份不可变快照，保存：

- 当前 server clients；
- `Arc<McpConfig>`；
- 模型可见的 `Vec<ToolInfo>`；
- `(server, tool) -> PreparedMcpCall` 映射。

所以同一 step 的 tool spec、tool search 和执行路由看到的是同一份目录。

#### 场景 C：模型调用一个 deferred MCP tool

如果工具被标成 deferred，模型先看到 `tool_search`：

```text
core/src/tools/handlers/tool_search.rs:54
  ToolSearchHandlerCache::get_or_build
    ├─ 收集 registry 中 exposure.is_deferred() 的工具
    ├─ 用 search_info 构造 BM25 documents
    └─ 返回 ToolSearchHandler

模型 → tool_search(query)
  → BM25 排名
  → 返回 LoadableToolSpec
  → 下一轮模型调用具体 tool
```

具体工具进入执行后：

```text
core/src/tools/handlers/mcp.rs:196
  McpHandler::handle_call
    ├─ session.prepare_mcp_call(tool_info)
    │    └─ 从当前 runtime 绑定精确 client/catalog
    ├─ core/src/mcp_tool_call.rs:handle_mcp_tool_call
    │    ├─ app policy / MCP approval
    │    ├─ PreparedMcpCall::call_with_preparation
    │    └─ 结果转换成 ToolOutput
    └─ 回到 §7 的结果回灌
```

#### 场景 D：模型请求安装一个 plugin / connector

模型只能提出请求，不能静默安装：

```text
core/src/tools/handlers/request_plugin_install.rs:55
  RequestPluginInstallHandler::handle_call
    ├─ 必须是 root thread
    ├─ candidate 必须来自 discoverable / recommended 列表
    ├─ 生成 request_plugin_install_<call_id>
    ├─ request_mcp_server_elicitation(codex_apps, ...)
    ├─ 用户 Accept？
    ├─ verify_request_plugin_install_completed
    ├─ connector 则 merge_connector_selection
    └─ 返回 { completed, user_confirmed, .. }
```

如果用户选择永久拒绝，还可以把 disabled suggestion 写入配置，避免以后重复推荐。

### ③ 关键设计

#### a. MCP runtime 是 thread 级“可替换发布”，binding 是 step 级快照

`McpRuntime`（`codex-mcp/src/runtime.rs:102`）保存一个
`ArcSwap<PublishedMcpRuntime>`。配置、auth、environment selection 或 plugin selection 变化后，
core 计算新的 `McpRuntimeInput` 并调用 `replace`。

发布过程中：

```text
McpRuntimeInput
  → McpConnectionSet::new(previous, ...)
  → 可复用的 client / 可缓存的 catalog
  → PublishedMcpRuntime
  → current.store(...)
```

`previous` 允许连接复用，但只有连接配置、启动状态、认证信息都匹配时才复用。
改变 startup timeout 不应杀死已经 ready 的连接；认证账号变化则必须重新判断 client
是否还能代表当前身份。

`McpBinding` 不是 runtime 的 live view，而是该 runtime 某一刻的冻结版本。原因和第二幕
工具表冻结一致：模型已经看到一组 schema，host 必须用同一组目录和权限执行对应调用。
若期间 server 刷新，旧调用仍能完成，新 step 才看到新 catalog。

#### b. Eager / lazy startup 是按会话类型选择的

`McpStartupPolicy` 有两个值：

- `Eager`：runtime 发布时启动 eligible server；
- `LazyWhenCached`：有缓存工具定义时先发布 catalog，首次真正需要时再启动。

普通 root session 使用 eager；subagent 使用 lazy cached。部分配置还允许
`startup_readiness = Catalog`，连接尚未 ready 时仍可用缓存目录让模型规划。

这不是简单省启动时间。MCP server 可能是本地进程、stdio、HTTP 或 remote executor 服务；
启动失败不应让整个 thread 无法显示能力，真正调用时的 `wait_until_ready` 仍会等待 target
server 并可能返回失败。

#### c. ToolInfo 同时需要“模型名”和“协议原名”

MCP server 返回的工具名不可信且可能冲突：大小写、长度、namespace、不同 server 同名、
plugin 与 app 重名都可能破坏 Responses API 的工具表。

`ToolInfo` 因此保留两套身份：

- `server_name` + `tool.name`：协议调用使用的原始身份；
- `callable_namespace` + `callable_name`：模型看到的稳定、唯一、有长度上限的名字。

`normalize_tools_for_model_with_prefix`（`codex-mcp/src/tools.rs:107`）会：

- 清理非法字符；
- 可配置历史 `mcp__` prefix；
- 去重相同 raw identity；
- 对 namespace / tool 冲突追加 hash suffix；
- 把模型名控制在 128 bytes 内。

执行时用 `ToolInfo` 内的原始名字回调 server。模型名字只是路由 UI，不是外部协议事实。

#### d. `defer_loading` 把“可发现”与“已展开 schema”分开

工具很多时，每个 function schema 都进入初始 prompt 会让输入巨大，还会频繁改变前缀。
ToolExposure 因此区分 direct / deferred / code-mode-only。

deferred tool 仍在 registry 中，普通模型看不到它的完整 schema，只看到 `tool_search`。
`tool_search`：

1. 对每个 deferred 工具建立搜索文档；
2. 用 BM25 搜索当前 query；
3. 返回最多 N 个 `LoadableToolSpec`；
4. 模型在后续 step 调用具体工具。

这是“结构化目录 + 轻量检索”的另一种应用，和 §11 的 skill 选择同属词法检索路线。
它减少常驻上下文，但多一次搜索 step；对高频核心工具仍应 direct exposure，避免每个 turn
都先搜一次。

#### e. 执行绑定到 PreparedMcpCall，而不是只绑定 tool name

真正调用前，`McpRuntime::prepare_call` 会把 advertised `ToolInfo` 解析成
`PreparedMcpCall`。后者保存：

- 精确 client；
- server metadata；
- catalog snapshot revision；
- timeout；
- permission profile；
- plugin / Apps 归属；
- tool annotations。

`PreparedMcpCall::call_with_preparation`
（`codex-mcp/src/binding.rs:304`）在 catalog snapshot 保护下准备并发送请求。这样即使
runtime 在请求途中刷新，调用也不会突然切到另一个 server 或另一份 schema。

审批仍然复用第二幕：server 的 approval mode、App tool policy、plugin selected 状态和
用户配置共同决定是否需要 elicitation。MCP 工具没有绕过 §8 的权限链，只是多了 server 级
和 connector 级策略输入。

#### f. MCP 认证状态与“工具级登录”分开

MCP 支持 stdio / HTTP 等 transport，认证可能来自：

- 配置或环境中的 bearer token；
- streamable HTTP OAuth；
- Codex 账号 / Apps connector 的 hosted auth；
- 特定 entitlement 注入的 trusted access context。

`mcp/auth.rs` 先计算每个 server 的 auth state 和 OAuth support。OAuth scopes 的优先级是：

```text
explicit scopes
  > configured scopes
  > discovered scopes
  > empty
```

如果 server 支持 OAuth，回调 URL 可按 server 区分；如果只是一个 bearer env var，
则不必进入 OAuth 流程。

`UseXaa` 是更窄的企业托管授权入口：只有 feature 开启且存在
`mcp_enterprise_managed_auth` 时，MCP config 才真正设置 `xaa_enabled`
（`codex-rs/core/src/config/mod.rs:1781`）。配置校验还要求该 opt-in 来自非 project
配置层或 managed requirement（`codex-rs/config/src/mcp_ema.rs:321`），避免仓库里的
`.codex/config.toml` 自行把账号切到企业授权模式。

工具已经发出后才发现 connector 需要 auth 时，`auth_elicitation` 会为原始 `call_id`
构造登录 elicitation。用户完成授权后，结果以“auth elicitation completed”返回，而不是
生成一条普通业务成功。这样模型知道它还没看到真实工具结果，runtime 也能按新 auth generation
刷新 catalog。

#### g. MCP resources 与 tools 是两种不同的数据面

Tool 表示“执行一个动作”，resource 表示“读取一份由 server 管理的内容”。两者不能合并：
server 可能只允许列 schema、读文件，不允许任意函数调用；也可能把私有数据作为 resource
暴露，而把写入动作作为 tool。

core 为 resource 提供：

- `list_mcp_resources`；
- `list_mcp_resource_templates`；
- `read_mcp_resource`。

`McpResourceClient`（`codex-mcp/src/resource_client.rs:279`）支持按 server 列一页 resource，
返回 `next_cursor`，再按 server + URI 读取内容。Apps 还有专用 resource 列表参数，
因为 plugin-service 需要每次请求都携带相同的 `mimeType` 过滤条件。

resource 的读取权限仍绑定当前 published runtime；连接被移除或 auth 变化后，旧 client
不能继续读取。读取结果可能包含文本或 blob，最终仍作为不可信外部内容进入当前历史。

#### h. Event stream 是长连接订阅，不是普通 tool result

hosted MCP server 还能声明 event。流程分两步：

```text
McpResourceClient::list_events
  → events/list
  → event name / delivery / input schema / payload schema

McpResourceClient::open_event_stream
  → 独立 MCP client
  → events/stream
  → notifications/events/active
  → notifications/events/terminated
```

`McpEventStreamManager`（`ext/mcp/src/stream_manager.rs:62`）按
`(thread_id, subscription_id)` 管理每一条订阅。每个 stream 有独立 worker，
激活超时 90 秒，update queue 容量 64。取消 stream 时不等待队列腾出空间，
这样可以避免用户退出或切换账号时被满队列阻塞。

access change、server removal 和 thread shutdown 都会取消对应 stream。stream 的每个
notification 带 `stream_attempt_id`，避免旧订阅结束事件误关联到新订阅。

这类事件不能被当作一次普通 MCP tool call 持久化；它属于 thread 运行中的事件源，
需要由上层消费者决定哪些内容要转成 UI 更新或新的模型输入。

#### i. Plugin 是声明式包，不是动态链接库

`PluginManifest`（`codex-rs/plugin/src/manifest.rs:8`）的核心结构是：

```text
PluginManifest {
  name, version, description, keywords,
  paths {
    skills,
    onboarding_skill,
    mcp_servers,
    apps,
    hooks,
  },
  interface,
}
```

manifest 可以内联 MCP server 配置，也可以指向某个文件；apps 指向 connector 声明；
hooks 指向文件或内联事件配置。`interface` 只放显示名、描述、图标、截图等 UI 元数据。

它没有 Rust trait 入口，也没有“加载后执行 init()”的插件 ABI。plugin loader 最终产出
`LoadedPlugin`：

```text
LoadedPlugin {
  skill_roots, skill_discovery_mode,
  mcp_servers,
  apps,
  hook_sources,
  ...
}
```

然后 skill / MCP / apps / hooks 分别交给各自子系统。plugin 是能力包的**装配清单**，
不是一个拥有主循环的新进程。

plugin 的分发生命周期由 `PluginsManager` 和 `PluginStore` 管理：

- marketplace 清单可以从 local、Git、npm 来源解析；
- 安装时 materialize 到 `CODEX_HOME/plugins/.../<version>`，同时保存 active version；
- remote plugin 保留 backend id 与本地安装 metadata，认证变化会触发账号隔离的 cache refresh；
- curated marketplace 有启动同步、Git/HTTP fallback 和本地快照；
- upgrade/remove 更新 active version 与配置，不直接改正在执行的 `McpBinding`。

marketplace 还能声明 install policy（不允许 / 可安装 / 默认安装）与 auth policy
（安装时或使用时认证）。推荐 plugin 会与已安装、被用户禁用的候选去重，再通过
`request_plugin_install` 暴露给模型。

#### j. Plugin 的可执行面被压缩到 MCP 和 hooks

plugin 不提供任意 native code 注入点，但它仍然可以影响动作执行，主要经过两条路：

1. MCP server：server 自己提供 tools，执行仍走 MCP 审批和 sandbox；
2. hook：在生命周期事件上调用命令或 MCP tool。

executor 发现的插件目前没有签名信任链，因此 `executor_plugin_hook_sources`
（`core-plugins/src/executor_hooks.rs:29`）只接受明确 allowlist 中的 cleanup hooks。
代码里保留了 `FIXME`，说明这是等待签名能力的临时收紧，而不是长期“所有插件 hook 都自由执行”。

本地安装的 plugin 也由 marketplace policy、requirements、trusted roots 等配置层约束。
因此“声明式”不等于“完全没有副作用”；准确说法是：plugin 自己不接管 Codex 执行，
它声明的 MCP/hook 动作仍然落到已有安全链。

#### k. Apps 是受账号和工作区约束的 MCP 工具集

当 `Feature::Apps` 开启，`HostedPluginRuntimeExtension` 向 runtime 贡献 host-owned
`codex_apps` MCP server。这个 server 在后端连接 GitHub、Drive、Slack 等 connector，
并把 connector 能力投影成 MCP tools。

因此 Apps 不是另一套 tool protocol；对模型来说它是 MCP 工具，区别在于：

- server 是 host-owned；
- 工具属于 connector；
- 可见性和审批由 `AppToolPolicy` 决定；
- catalog 按账号 / workspace 隔离并缓存。

`AppToolPolicyEvaluator::policy`（`connectors/src/app_tool_policy.rs:52`）综合：

- 用户 apps 配置；
- managed requirements；
- connector 是否启用；
- tool 显式 enabled；
- `destructiveHint` / `openWorldHint`；
- tool / app / link 级别的 approval mode。

managed approval 优先级高于本地 tool config；若 app 被禁用，后续 tool 级 enabled=true
也不能重新打开。

#### l. Connector catalog 是账号隔离的共享缓存

Connector/Apps 工具列表从 ChatGPT 后端获取。目录缓存 key 包含：

- base URL；
- account id；
- ChatGPT user id；
- 是否 workspace account。

目录 TTL 是 1 小时，并有进程内缓存和 best-effort 磁盘持久化。`ConnectorRuntimeManager`
再按同一身份共享最新工具 snapshot；不同账号或 workspace 不共享可执行 catalog。

这里的复杂度来自“app 定义”和“工具定义”不是同一份数据：app directory 提供展示与
install 信息，connector runtime 提供 tools；工具 policy 又在读取时应用，而不是把
用户配置永久写进共享缓存。

#### m. Plugin / connector 安装必须经过显式用户确认

`request_plugin_install` 不是普通“下载并启用”工具。它先验证：

- 当前是 root thread；
- 请求的 id 确实出现在本 turn 的可发现列表中；
- 推荐插件的 metadata 仍属于当前 auth / config；
- 客户端支持这个安装流程。

然后通过 `codex_apps` 发 elicitation，让用户 Accept / Decline / Cancel。
只有 Accept 后才继续验证安装结果：

- connector：重新发现 accessible connectors；
- plugin：检查安装后的 plugin / MCP / app 能力是否真的可用。

`completed` 与 `user_confirmed` 分开返回，避免 UI 把“用户点了同意”误报成“安装已经生效”。
这也是为什么 plugin install handler 不是 `supports_parallel_tool_calls`：它包含用户交互和
全局配置变更，不能与其它工具并发放行。

#### n. 外部上下文会反向影响 memory 信任级别

MCP / Apps 返回的是外部内容，可能包含 prompt injection。除了审批和 sandbox，
系统还有一条与 §12 交叉的边界：当配置开启
`memories.disable_on_external_context`，来自会污染外部上下文的 server / item 可以把
thread 的 memory mode 标记为 polluted。

`PreparedMcpCall::server_pollutes_memory` 决定一次 MCP 调用是否触发这个标记；
外部 connector auth flow 和外部 response item 也有对应检查。结果不是删除当前 history，
而是阻止这条 thread 被当成干净经验自动写进长期 memory。

#### o. 两类协议版本开关影响连接和 Apps

`Mcp20260728` 控制普通 MCP 协议模式，`CodexAppsMcp20260728` 单独控制 host-owned Apps
server。两者分开是因为 Apps 走可信 host 路线，升级节奏和普通第三方 server 不同。

协议切换发生在建立连接之前，由 config 生成 protocol mode。已有 ready client 只有配置
兼容时才可复用，不能把旧协议 client 悄悄接到新模式下。

### ④ 取舍

**为什么 MCP 用 immutable binding，而不是每次调用都查最新 catalog？**

模型看到 schema 后，host 必须用与 schema 一致的参数和权限执行。若中途刷新，
工具可能被删除、换 server 或改变 schema，调用会变得不可解释。binding 把一次 step 的
目录和执行句柄冻结在一起。

代价是刷新后旧 binding 还持有旧连接，资源释放要等最后一个引用消失；同时缓存键必须包含
auth、environment、plugin selection 等所有影响可执行性的输入，漏一项就会错误复用。

**为什么需要 deferred tools 和 `tool_search`？**

不延迟加载，MCP/Apps 工具很容易在初始 prompt 里占掉数万 token；每次都延迟，常用工具又要
多一步搜索。Codex 允许不同 exposure 并存：核心工具 direct，长尾工具 deferred。

代价是模型必须理解搜索结果中的 loadable tool，并承担一次额外推理轮。工具描述质量、
namespace 冲突和搜索排名都会影响可达性。

**为什么 plugin 不直接执行原生代码？**

原生代码插件需要版本 ABI、权限声明、签名、崩溃隔离和跨平台打包。当前 manifest 把这些
能力拆给更窄的接口：skill 是说明，MCP 是外部协议，hooks 是受限事件，apps 是 connector。
安全边界更容易落到现有工具审批与 sandbox 上。

代价是某些深度 UI 集成无法只靠 plugin 完成，必须进入 core/app-server；plugin 能表达的
自动化也比任意脚本更窄。

**为什么 Apps 不直接走 web search？**

GitHub、Slack、Drive 的私有数据和动作需要账号身份、workspace 权限、结构化 schema 和
审批。Web search 面向公开网页，无法可靠代表“当前用户在某个私有资源上的权限”，也不能
安全执行写操作。

代价是 Apps 需要额外的 connector auth、目录缓存、tool policy 和 MCP server 生命周期；
但它把私有数据访问收敛到明确的身份和权限模型，而不是让模型从网页文本猜。

**为什么外部工具输出不能直接成为 memory？**

外部内容可以要求模型忽略规则、泄露信息或执行下一步动作。即使工具调用本身已获批准，
输出仍是不可信数据。把它自动写进跨会话 memory，会让一次注入长期污染后续任务。

所以 Codex 在 MCP metadata、external context item 和 memory mode 之间建立显式标记：
外部来源可以参与当前任务，但需要额外信任条件才值得进入长期记忆。

回到主流程：§13 完成后，一个扩展能力已经可以来自本机文件、远端 executor、云 skill、
plugin 包、MCP server 或 Apps connector。它们都通过同一组 contributor 接点进入
thread / turn，并在执行时回到第二幕的审批与结果回灌链路。

---

## 幕末收获

读完这一幕，你应该能：

- 说清 resume 到底恢复了什么
- 解释 contributor 框架解决了什么问题、放弃了什么
- 说清 skill 为什么是“目录 + 指令包”，以及 host / executor / cloud 的读取边界
- 解释 memory 为什么拆成读摘要、Phase 1 抽取和 Phase 2 consolidation
- 判断一个新能力该用哪种扩展形态（contributor / skill / plugin / MCP / App）
- 跟踪一次外部工具调用从 server contribution、tool binding 到审批和结果回灌的路径

## 下一幕引子

能力齐了，但会话会越来越长，上下文窗口迟早会满；单个 agent 也不够用。
下一幕解决"长会话"和"规模化"。

→ [第四幕：长会话与规模化](act-04-long-session-and-scale.md)
