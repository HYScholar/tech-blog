---
title: "279 行代码的 Agent 编排方案：Hermes Kanban Swarm 深度解析"
date: "2026-07-12"
tags: ["Hermes", "Kanban", "Swarm", "多Agent", "Agent编排", "SQLite", "开源"]
author: "HYScholar"
---

# 279 行代码的 Agent 编排方案：Hermes Kanban Swarm 深度解析

如果你要做多 Agent 协作，你的第一反应是什么？

加一个新调度器。定义一个新运行时。建一套状态管理。LangGraph 这么做了，CrewAI 这么做了，AutoGen 也这么做了——每个方案都是上万行代码的新抽象层。

Hermes Agent v0.15+ 的 Kanban Swarm 给出的答案是：**279 行 Python。** 没有新调度器，没有新运行时，没有新数据库。所有逻辑跑在一张 SQLite Kanban 看板上。

这不是省事。这是对一个更根本问题的回答：Agent 编排到底需要什么？

## 不做第二个调度器

`kanban_swarm.py` 的模块文档开头只有一句话：

> This module intentionally does not introduce a second scheduler. It writes a small task graph into the existing Kanban kernel.

这句话是整个设计的基石。

行业中几乎所有人都选择了"向上构建"：在已有 Agent 之上叠加新的协调层。LangGraph 的 StateGraph，CrewAI 的 Manager-Worker 层级，AutoGen 的对话协议，每个都是独立运行时。问题是，多一个抽象层，就多一层状态同步、多一层故障恢复、多一层调试困难。

Hermes 的答案不同。它问的是：已有的 Kanban board 能不能直接承载协作图？

Kanban 本身就是状态机。rows 是工作项（tasks），columns 是状态转换（todo → ready → running → done），edges 是依赖关系。这套抽象来自 1970 年代丰田的制造看板，比 AI Agent 早了五十年。任务分解、并发执行、状态追踪、故障恢复——协调问题的本质是通用的。而这些问题恰好是 Kanban 本来就该干的活。

## Swarm 图：并行 + 门控 + 合成

Swarm 的拓扑结构极其简单，只有四种角色：

```
Root（共享黑板）
├── Worker 1（并行）
├── Worker 2（并行）
└── Worker N（并行）
    └── Verifier（门控）
        └── Synthesizer（最终合成）
```

**Root 卡**是锚点。创建后立即标记完成——这样 workers 可以立刻开始。后续所有跨 worker 的信息共享（研究结果、代码片段、设计决策）都写成这张卡的 structured JSON comments。这就是 Swarm 的"黑板书"。

**Worker 卡**各自跑在独立的 Hermes Agent 进程里，有各自的 profile、工具链和 skills。三个 researcher 并行调研同一主题的不同角度，两个 coder 同时审查代码的不同层面。互不干扰。

**Verifier 卡**是门控。它必须等到所有 workers 完成才可执行（通过 parent→child 依赖自动推进），且必须返回 `metadata: {"gate": "pass"}` 才能放行 Synthesizer。不通过就 block，同时说明缺什么。根据对 17 种多 Agent 拓扑的分析，37% 的系统故障来自协调失败，21% 来自验证缺口。Verifier gate 用一个简单的状态转换堵住了这两个最常见的漏洞。

**Synthesizer** 拿到所有 workers 的产出和 verifier 的放行信号后，产出最终交付物。

用 CLI 触发只需要一条命令：

```bash
hermes kanban swarm "Audit our API surface for security regressions" \
  --worker researcher:"Scan endpoints and dependencies":web \
  --worker coder:"Check auth middleware implementation" \
  --worker coder:"Review rate limiting and input validation" \
  --verifier reviewer \
  --synthesizer writer
```

## SQLite 黑板书：最简单的分布式系统

Swarm 所有状态都落在 SQLite 里。这是整个方案最让人意外也最高明的部分。

**并发靠 CAS，不靠锁。** Dispatcher 认领任务时用 `UPDATE tasks SET status='running', claim_lock=... WHERE status='ready' AND claim_lock IS NULL`——Compare-and-Swap。SQLite 的 WAL 模式自动序列化写入者，同一任务只有一个人能认领成功。失败的看到 0 affected rows，直接跳过。没有重试循环，没有分布式锁。

**"黑板书"就是 JSON.stringify 写进 comment 列。** 没有 gRPC，没有消息队列，没有共享内存。Worker 调用 `kanban_comment(task_id, body={...})` 就是一次黑板写入；下游 worker 通过 `kanban_show(task_id)` 读取 parent 的 handoff 就是一次黑板读取。这是整个系统里最诚实的部分。你可以说它简陋，但没有更好的方式保证跨进程、跨崩溃、跨重启的信息不丢。

**持久化等于可生存。** 所有任务状态、执行历史（task_runs）、事件日志（task_events）、评论和黑板（task_comments）都在 SQLite 行里。Dispatcher 挂了？重启后 reclaim 过期认领。Worker 进程被杀？下次 tick 检测到 PID 消失，自动 re-queue。人类想介入？在同一个 board 上 comment 或 unblock，跟 Agent 没有任何区别。

## 对比赛场

当 OpenAI 发布 Swarm 框架时，社区最初很兴奋。轻量级，零依赖，Agent 间通过函数调用切换。然后有人跑了基准测试：随任务复杂度增加，准确率从 84% 一路崩塌到 0%。原因很简单，没有全局状态跟踪。Agent A 切换到 Agent B 后，A 的上下文全部丢失。

这不是 OpenAI Swarm 独有的问题。LangGraph 的状态图需要开发者手动定义每个节点和边，复杂任务下状态爆炸在所难免。CrewAI 的 Manager-Worker 固定层级在需要灵活拓扑时反而成了约束。AutoGen 的对话协商协议优雅但调试起来痛苦——当三个 Agent 各说各的，你很难还原最后到底是谁决定了什么。

Kanban Swarm 用三个很直接的手段避开了这些坑：

状态不丢。Agent 之间传递的不再是"上下文窗口"，而是数据库中的 structured metadata。SQLite 行是持久的，crash 之后数据还在。

人在环中。Board 对人类和 Agent 是同一块板子。reviewer 说"不通过，缺 XX"，writer 看到 block，补上，人类 unblock，writer 重新运行。人和 Agent 的交互不需要桥接层。

故障可恢复。crash 后 reclaim，连续失败触发 circuit breaker auto-block，无限 unblock-reblock 循环由 recursions counter 截断到 triage。这还不是全部——respawn 守卫还会检查最近的配额/认证/PR 状态，防止在同一个坑里反复跌倒。

## 实际场景比你想的多

别看 Swarm 代码只有 279 行，设计规范里列出了 8 种协作模式，覆盖了大部分实际需求。

Fan-out 是最直接的：同一个主题，5 个 researcher 并行调研不同角度，1 个 synthesizer 汇总。Pipeline 适合有明确工序的任务——scout 收集素材，editor 筛选，writer 成稿，一条串行依赖链。Quorum 模式用在需要多角度验证的场景：3 个 reviewer 独立审查同一段代码，1 个 aggregator 按多数意见产出合并结果。Human-in-the-loop 则是为模糊决策留的口子——worker 不知道怎么选就 block(kind="needs_input")，人类在 board 上 comment 给方向，unblock 后 worker 继续。

这篇文章碰巧是个活例子。Orchestrator 把任务拆成 Researcher → Writer → Reviewer → Publisher 四个子任务，各自配上对应的 skills，形成串行依赖链。Writer 拿到的不是模糊的"调研一下 Kanban Swarm"，而是一份 17KB 的结构化笔记，里面标好了源码行数、竞品数据、设计分析。不需要重新搜索，不需要重复推理——拿起来就能写。

## 结语

1970 年，丰田工程师设计看板系统来协调汽车生产线上的物料流和人力分配。核心问题是：一个任务依赖另几个任务完成；并发执行需要避免冲突；瓶颈出现时需要有人看到并处理。

2026 年，AI Agent 面临完全相同的协调问题。Hermes Kanban Swarm 用 279 行代码和一张 SQLite 看板说明了一件事：答案不在"再加一层"里。答案在 1970 年那张看板上。

---

*延伸阅读：*

- [Hermes Agent Kanban 官方文档](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban) — 978行完整功能参考
- [Hermes Agent GitHub](https://github.com/NousResearch/hermes-agent) — 源码（kanban_swarm.py 279行，kanban_db.py ~8750行）
- [Kanban Swarm 设计规范](https://github.com/NousResearch/hermes-agent) — 8种协作模式完整定义（见仓库 docs/hermes-kanban-v1-spec.pdf）
- [Magnus919: The Smartest Agent Orchestration Framework Doesn't Have a Scheduler](https://magnus919.com/2026/05/the-smartest-agent-orchestration-framework-doesnt-have-a-scheduler/) — 第三方深度分析
- Kanban 看板方法起源：[Toyota Production System](https://en.wikipedia.org/wiki/Kanban) (1970s)
