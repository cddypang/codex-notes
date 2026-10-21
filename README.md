# Codex 源码拆解笔记

## 这是什么

一份**围绕"一个编码智能体是怎么做出来的"的源码阅读笔记**。

想回答的问题只有一个：**Codex 由哪些能力构成，每个能力在代码里是怎么落地的。**

围绕这个问题，笔记分两条线展开：

| 线索 | 文件 | 回答什么 |
|---|---|---|
| **横向** | [codex-agent-architecture.md](codex-agent-architecture.md) | 有哪些能力（RAG / Memory / Skill / Plugin / MCP……），各自实现在哪、怎么实现 |
| **纵向** | [act-00](act-00-prerequisites.md) ~ [act-04](act-04-long-session-and-scale.md) | 按"从 0 到 1 长出一个 agent"的顺序，逐层拆开代码 |

> 这不是官方文档，也不是 API 手册，而是逐文件读源码后的个人整理。
> 结论以代码为准，代码没看到的地方会明确标注"未核实"。

## 基准版本

所有代码引用（`文件:行`）都以这个提交为准：

```
仓库    https://github.com/openai/codex
commit  8c3612fb638d356579c446de68421288164484dc
短哈希  8c3612fb63
日期    2026-09-30 02:19:47 +0000
标题    Use `memrchr` to find newlines in reverse JSONL scans (#49444)
```

**两点说明**：

- 拆解时工作区存在未提交改动（`codex-rs/Cargo.toml`、`codex-rs/Cargo.lock`，
  以及一个未跟踪的 `rusty-v8-msvc/` 目录）。这些都属于构建配置，**不涉及本次拆解覆盖的
  `core` / `codex-api` / `ext` / `app-server` / `tui` 等源码**。
- 上游持续演进，行号会随时间漂移。引用时建议用**函数名**而不是行号定位。

## 文档结构

### 纵向：四幕

按"一个 agent 从最小闭环长到成熟"的顺序组织，越往后依赖越多。

| 文档 | 行数 | 内容 | 状态 |
|---|---|---|---|
| [act-00-prerequisites.md](act-00-prerequisites.md) | 892 | **第零幕 · 预备知识**：进程与角色（含 daemon 职责、`--no-daemon` 行为、`codex agents` / `queue`）、基础结构（`ThreadManager` / `CodexThread` / `Session`……）、设计原则与前置知识 | ✅ 完成 |
| [act-01-minimal-loop.md](act-01-minimal-loop.md) | 1701 | **第一幕 · 最小闭环**：§1 启动与线程模型、§2 turn 生命周期、§3 模型接入、§4 上下文组装、§5 world state 与增量渲染 | ✅ 完成 |
| [act-02-tool-loop.md](act-02-tool-loop.md) | 1208 | **第二幕 · 工具循环**：§6 工具抽象、§7 工具循环与并发、§8 审批与安全边界 | ✅ 完成 |
| [act-03-memory-and-extensions.md](act-03-memory-and-extensions.md) | 234 | **第三幕 · 记忆与扩展**：§9 持久化与恢复、§10 Contributor 框架、§11 Skill、§12 Memory、§13 MCP + Plugin + Apps | 📝 大纲 |
| [act-04-long-session-and-scale.md](act-04-long-session-and-scale.md) | 168 | **第四幕 · 长会话与规模化**：§14 Compaction、§15 多智能体、§16 Code Mode 与 Realtime | 📝 大纲 |

节号 **1–16 跨四幕连续编号**，所以「第 N 节」全局唯一。

### 横向：能力清单

[codex-agent-architecture.md](codex-agent-architecture.md)（691 行）是另一条线——
按"能力"而不是"层次"组织，适合**当速查表用**，不适合顺序阅读：

- 2. 能力清单（哪些有、哪些没有）
- 3. RAG 的真实情况（**没有向量检索**，以及五个替代方案）
- 4–7. Memory / Skill / Plugin / MCP 各自怎么实现
- 9. Contributor 是什么（13 个 trait）
- 10. 核心智能体模块地图
- 11–12. turn / thread / session 的边界，以及 resume / clear / fork 行为

## 怎么读

推荐顺序：**act-00 → act-01 的 §1 和 §2 → 想深入哪块就跳哪块**。

每一节都固定按四段写：

```
① 问题       没有这个东西会怎样
② 代码入口   从哪个文件哪一行开始读，调用链是什么
③ 关键设计   核心 trait / 数据结构 / 算法
④ 取舍       为什么这么做，代价是什么，和替代方案比
```

第 ④ 段通常是最有价值的部分——比如"为什么 contributor 是编译期 trait 而不是运行期插件"、
"为什么 world state 用 merge patch 而不是全量"、"为什么 memory 用 LLM 抽取而不是向量检索"。

`② 代码入口` 里的每一行都是 `文件:行 → 函数名` 的形式，按顺序在编辑器里打开就能走到底。

## 已经得到的几个结论

如果只想看结论，这几条是整份笔记的骨架：

1. **所有 LLM 能力都挂在 contributor 插槽上。** core 只认识 13 个 trait，不认识 memory / skill /
   MCP / plugin。加一个新能力的标准动作是新建一个 `ext/xxx` crate，实现需要的 trait，
   然后在 `app-server/src/extensions.rs` 的 `thread_extensions()` 里加一行 `install`。
2. **没有向量 RAG。** 搜索用 `embedding` 全仓库只命中注释。替代品是结构化的上下文注入 +
   LLM 驱动的离线记忆固化 + 轻量词法检索（BM25 / n-gram / `tool_search`）。
3. **上下文是"增量追加 + 差异渲染"的。** 历史不重写，新内容只能追加；
   `WorldState` 只把"和上一份快照相比变化了的部分"发给模型，为的是 prompt cache 能命中。
4. **能力是声明式的，不是探测出来的。** `ProviderCapabilities` 是上界，
   `supports_codex_backend_routes()` 决定能不能走私有端点。不满足时工具根本不出现在工具表里，
   而不是调用后报错。
5. **执行安全是重投入。** 沙箱、`execpolicy`、审批、Guardian 构成四层，
   而通用 agent 框架在这一层几乎空白。

## 约定

- 所有路径都是**仓库根的相对路径**，不带 `codex-rs/` 前缀的属于仓库其他部分。
- 跨文档引用会写成「第一幕第 2 节」或者带链接，避免和局部编号混淆。
- 引用的注释原文一律用引用块（`>`）标出，不改写。
- 没联网核实的判断（比如第三方框架的当前状态）会在正文里说明。
- **不展开的范围**：各平台沙箱的内部实现、app-server 协议细节、SDK、TUI 渲染、构建系统。
  它们会在流程里被提到，但只讲"位置和作用"。

## 这个仓库

本目录是**独立的 git 仓库**，和上层的 `openai/codex` 源码仓库无关——
后者只是阅读对象。这样笔记的提交历史不会被源码仓库的变更淹没。
