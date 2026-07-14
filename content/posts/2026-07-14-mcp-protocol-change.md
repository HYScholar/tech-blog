---
title: "MCP 协议史上最大变更——2026年7月28日倒计时，4356个服务器仅1个做好准备"
date: 2026-07-14
tags: ["MCP", "Agent", "协议", "Anthropic", "开源", "工程化"]
categories: ["技术"]
author: "HYScholar"
draft: false
---

---

## 一、MCP 是什么（用两段说清楚）

MCP 的全称是 Model Context Protocol。Anthropic 在 2024 年 11 月把它开源，思路很简单：给 AI 应用和外部工具之间定义一个标准通信协议，就像 USB-C 之于外设。

没 MCP 的时候，你想让 Claude 查数据库、Cursor 读文件系统、ChatGPT 调 API，每个组合都得写一套胶水代码。MCP 想要的是——Server 提供能力（Tools/Resources/Prompts），Client 用 JSON-RPC 2.0 发请求，一次集成到处可用。

两年下来，数据挺夸张：10,000+ 公共 MCP 服务器，月 SDK 下载量 97M+。2025 年底 Anthropic 把协议捐给了 Linux Foundation 下的 Agentic AI Foundation (AAIF)，治理权从一家公司手里交了出来。国内生态也跟上了，CSDN、知乎上 MCP 入门教程一搜一大把。

背景交代完了。说正事。

---

## 二、这次变更到底改了什么

MCP 从有状态协议变成了无状态协议。听起来像架构课术语，但这事对你写的代码有直接后果。以前，Client 和 Server 通信之前得先「握个手」：

**旧模式（2025-11-25）：**
```
POST /mcp HTTP/1.1
Mcp-Session-Id: 1868a90c-3a3f-4f5b
Content-Type: application/json

{"jsonrpc":"2.0","id":2,"method":"tools/call",
 "params":{"name":"search","arguments":{"q":"otters"}}}
```

Client 先调 `initialize` 建 session，拿到一个 `Mcp-Session-Id`，后面每个请求都带着它。Server 在内存里维护这个 session 的上下文。

**新模式（2026-07-28）：**
```
POST /mcp HTTP/1.1
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: search
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

{"jsonrpc":"2.0","id":1,"method":"tools/call",
 "params":{"name":"search","arguments":{"q":"otters"},
           "_meta":{"io.modelcontextprotocol/clientInfo":{"name":"my-app","version":"1.0"}}}}
```

`Mcp-Session-Id` 没了。`initialize` 握手没了。协议版本、client 信息全塞进了每个请求的 `_meta` 字段。Server 不需要记住你是谁——每次请求自己带齐一切。

如果你在 nginx 后面跑了三个 MCP Server 实例做负载均衡，旧模式要求用 `ip_hash` 做 sticky session。一个 Client 的多个请求必须打到同一台机器上，因为 session 在那台机器的内存里。这意味着你不能随便扩缩容，一台机器挂了，上面所有 session 全丢。这是分布式系统里的经典死胡同。

新模式意味着你可以用 `least_conn` 或者 round-robin——随便哪台机器来处理请求都行，因为它不依赖本地内存里的 session。

### 完整的变更清单

这次 RC 由六个 SEP（Spec Enhancement Proposal）构成，每个都在拆掉有状态架构的某个零件：

| SEP | 改了什么 |
|-----|---------|
| SEP-2575 | 移除 `initialize`/`initialized` 握手，协议版本和 clientInfo 迁移到请求的 `_meta` 字段 |
| SEP-2567 | 移除 `Mcp-Session-Id` header，传输层 session 完全消除 |
| SEP-2243 | 新增 `Mcp-Method` 和 `Mcp-Name` headers，网关可以不解 JSON body 就能路由 |
| SEP-2549 | 查询结果支持 `ttlMs` 和 `cacheScope` 缓存元数据，类似 HTTP `Cache-Control` |
| SEP-414 | W3C Trace Context 标准化，OpenTelemetry 兼容 |
| SEP-2322 | Multi Round-Trip Requests 替代 SSE 长连接，客户端交互不再绑定到特定连接 |

此外，Extensions 框架正式成为一等公民。MCP Apps（SEP-1865）允许服务器提供 sandboxed iframe UI，Tasks 从实验性 API 重构为扩展，用 polling 模型替代 SSE 订阅。

工具 schema 也升了级（SEP-2106）：`inputSchema` 和 `outputSchema` 现在支持完整的 JSON Schema 2020-12，`oneOf`、`anyOf`、`$ref` 全可以用。

---

## 三、为什么整个生态都「没准备好」

回到开头那个数字。4356 个服务器，1 个合规。准确率约 0.023%。

理解这个数字需要看上下文。跑扫描的工具 `mcp-spec-check` 检测的是 2026-07-28 RC 规范——而这个规范还没正式发布。大部分服务器用的是 2025-11-25 的稳定版本，它们当然不会通过一个未来规范的检查。

社区也没被这个数字吓住。Hacker News 讨论里最高赞的评论大意是：「当然没有服务器会兼容一个还没发布的规范。」另一位用户更直接：「这不是 90% 的服务器会炸的故事，是 adoption baseline 的快照。」

真正有意思的数据在别的地方。同样的扫描显示，75.9% 的鉴权墙服务器已经发布了 RFC 9728 的 protected-resource metadata。社区在安全加固上的投入远超无状态迁移。为什么？因为 2026 年 4 月 OX Security 披露了一波 MCP 系统漏洞，影响了约 200,000 个实例——安全问题比 session 管理紧迫得多。

迁移确实有工作量，但并不像那个数字暗示的那么绝望。难度取决于你的 server 对 session 的依赖程度：

| Server 类型 | 迁移工期 | 典型改动 |
|------------|---------|---------|
| 无状态工具（文件读取、API 包装） | 0.5-2 天 | 升级 SDK、加 `Mcp-Method` header、添加 `/mcp/discover` endpoint |
| 有 session state 的 server | 10-15 人天 | 把 session context 外部化（Redis/DB），去掉内存 Map |
| 用了 experimental Tasks API | 需要重写 | polling 模型替代 SSE 订阅 |

而且时间窗口不紧。官方政策：deprecated 功能从 2026 年 7 月 28 日起存活至少 12 个月。这意味着旧 server 至少能跑到 2027 年 7 月——不是「7 月 28 日一到全挂」。

---

## 四、你现在该做什么

### 如果你是 MCP Server 维护者

三步走：

**1. 升级 SDK。** Python 用户 `pip install "mcp>=2.0.0"`（当前 beta：`mcp==2.0.0b1`），TypeScript 用户等 `@modelcontextprotocol/sdk` 的 RC 兼容版本发布。Tier 1 SDK 预计 7 月 28 日同步。

**2. 去掉 session。** 旧代码里常见的模式：
```typescript
// 旧：session 存在内存里
private sessions = new Map<string, SessionContext>();
async handleToolCall(request) {
  const session = this.sessions.get(request.sessionId);
  return this.executeWithContext(request, session);
}
```
改成显式 handle：
```typescript
// 新：state 通过参数传递
async handleToolCall(request) {
  const context = request._meta?.context;
  return this.executeWithContext(request, context);
}
```

**3. 加 discover endpoint。** 新规范要求 Server 响应 `GET /mcp/discover`，返回它支持的协议版本和能力列表。

### 如果你是 Hermes Agent 用户

Hermes 的 MCP 实现用的是 `mcp` Python 包。直接的影响：需要 SDK 升级到 ≥2.0.0（v2 是 2026-07-28 协议的唯一支持版本）。`mcp_servers` 的配置格式不受影响——`command/args`（stdio）和 `url/headers`（HTTP）的写法不变，transport 层的变更被 SDK 内部消化了。

旧 server 在 deprecated 窗口内继续工作。如果 `hermes mcp test` 开始报 protocol mismatch，就是你该升级 SDK 的信号。

### 负载均衡很简单了

以前要用 `ip_hash` 保证请求打到同一台机器：
```nginx
upstream mcp_backend { ip_hash; ... }
```

现在随便用：
```nginx
upstream mcp_backend {
    least_conn;
    server mcp-1.internal:3000;
    server mcp-2.internal:3000;
    server mcp-3.internal:3000;
}
```

甚至可以按 `Mcp-Method` header 做流量控制——网关不解 JSON body 就能路由，比之前快不少。

---

## 五、这次变更对 Agent 生态意味着什么

说真的，我对这次变更有种复杂的感觉。

一方面，无状态设计是对的。任何一个做过分布式服务的工程师都能理解——有状态协议和水平扩展天然冲突，这件事在 RESTful API 时代就被证明过了。MCP 用两年走到这一步，速度不算慢。

另一方面，0.023% 的合规率哪怕是「adoption baseline」，也说明了一件事：MCP 的服务器生态比 Protocol 规范年轻得多。很多社区服务器是周末项目，作者可能已经不管了。这次的变更不会把它们「炸掉」——12 个月的 deprecated 窗口给足了时间——但没人维护的服务器会慢慢烂掉，这是任何快速迭代的开源协议都躲不过的代价。

好消息是治理在理顺。AAIF 接管后搞了正式的 feature lifecycle policy：Active → Deprecated → Removed，最少 12 个月窗口。以后不会有突然的 breaking change，因为 SEP 要走到 Final 状态必须先落地 conformance suite 场景（SEP-2484）。Angie Jones 的说法很实在：「如果 MCP 试图拥有太多，它会变得更难实现和更难理解。如果它正确地拥有正确的事情，生态系统可以用更少的摩擦增长。」

MCP 正在从开发者玩具变成生产级基础设施。这个过程不优雅，但不优雅才是正常的。你能做的最聪明的事，不是等到 7 月 28 日看别人怎么办，而是提前升级你的 SDK，去掉 session，然后享受无状态架构带来的好处。

---

*参考来源：MCP 官方 RC 公告、AAIF 解读、mcp-spec-check 扫描数据、HN 社区讨论、Developers Digest 迁移指南、BOVO Digital 教程、Hermes Agent MCP 文档等。详见研究笔记 research-notes.md。*

---

## 延伸阅读

- [MCP 2026-07-28 RC 官方公告](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/) — David Soria Parra & Den Delimarsky，变更全貌和设计动机
- [MCP Draft 规范变更日志](https://modelcontextprotocol.io/specification/draft/changelog) — 逐条对照 `2025-11-25` 的所有变化
- [MCP Python SDK v2.0.0b1](https://github.com/modelcontextprotocol/python-sdk/releases/tag/v2.0.0b1) — 首个支持 2026-07-28 的 Python SDK 版本
- [AAIF 解读：MCP 正在长成](https://aaif.io/blog/mcp-is-growing-up/) — Angie Jones 谈治理、扩展框架和协议边界
- [Developers Digest 无状态迁移指南](https://www.developersdigest.tech/blog/mcp-stateless-migration-guide-2026) — 按 server 类型的迁移路线图
- [BOVO Digital 迁移教程](https://www.bovo-digital.tech/en/blog/tutorial-mcp-server-stateless-migration-2026) — 含完整 TypeScript + Python 代码
- [mcp-spec-check 工具](https://github.com/Roee-Tsur/mcp-spec-check) — 本文引用扫描数据的来源工具
- [Hacker News 讨论帖](https://news.ycombinator.com/item?id=48881009) — 「4,356 中仅 1 个合规」的原始讨论

> **审核通过** | Reviewer: Hermes Agent (reviewer profile) | 日期: 2026-07-14
> 修改记录：修正 Python SDK 版本号（mcp>=1.10.0→mcp>=2.0.0），新增 8 条延伸阅读链接
