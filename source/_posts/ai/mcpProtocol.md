---
title: MCP 最新版解读：无状态协议、扩展体系与 agent 能力全景
date: 2026-10-09 13:01:00
author: ws
description: 最新版 MCP 把协议核心改为无状态，并引入 MRTR 与官方扩展体系。本文讲清这次改版的关键变化，以及 MCP 配合 agent 的典型用法
categories: ["AI"]
tags: ["MCP", "Agent", "协议", "AI"]
cover:
---

## 引子

MCP（Model Context Protocol）发布至今不到两年，已经成为连接 AI 应用与外部系统的默认标准：Tier 1 SDK 每月下载量接近 5 亿，Python 和 TypeScript SDK 累计都突破了 10 亿次。2026 年 7 月 28 日发布的 `2026-07-28` 版本，是它诞生以来最大的一次改版——把协议核心从「有状态会话」改成「无状态请求」，并第一次建立了正式的扩展体系。

对 agent 开发者来说，这次改版有两个直接的结果：远端 MCP 服务可以像普通 HTTP 服务一样水平扩容；长任务、交互式 UI、流程知识这些能力都有了标准件。

这篇文章讲两件事：新版协议变了什么，以及 MCP 现在能配合 agent 做什么。

## 30 秒回顾

MCP 用 JSON-RPC 2.0 消息把三类角色连起来：

- **Host**：LLM 应用本身（Claude、ChatGPT、IDE 等），管理多个客户端实例；
- **Client**：宿主内部与某个 server 一一对应的连接器；
- **Server**：以工具、资源、提示词的形式提供能力。

Server 提供三类能力：`tools`（模型可调用）、`resources`（数据与上下文）、`prompts`（模板化工作流）；Client 提供 `elicitation`（代表用户回答问题）。官方把它比作 AI 应用的 USB-C 口：接一次，处处可用。

版本时间线：

| 版本 | 关键变化 |
| --- | --- |
| 2024-11-05 | 首个版本，stdio + HTTP+SSE |
| 2025-03-26 | Streamable HTTP 取代 SSE；OAuth 2.1 授权框架 |
| 2025-06-18 | elicitation；结构化工具输出；移除 JSON-RPC 批处理 |
| 2025-11-25 | Tasks（实验性）；sampling 支持工具调用；URL 模式 elicitation；CIMD；扩展概念 |
| 2026-07-28 | 无状态核心；MRTR；官方扩展体系；一批弃用 |

## 一、无状态：这次改版的主线

### 握手和会话都没了

旧版协议要先 `initialize` 握手、协商能力，HTTP 传输还会分配 `Mcp-Session-Id`。新版把这两样都删了：

- 每个请求自己在 `_meta` 里携带协议版本、客户端信息和能力声明；
- 服务端在每个响应的 `_meta` 里带上自己的身份；
- 版本不匹配时返回 `UnsupportedProtocolVersionError`。

```http
POST /mcp HTTP/1.1
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: search
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"tools/call",
 "params":{"name":"search","arguments":{"q":"otters"},
 "_meta":{
   "io.modelcontextprotocol/protocolVersion":"2026-07-28",
   "io.modelcontextprotocol/clientInfo":{"name":"my-app","version":"1.0"},
   "io.modelcontextprotocol/clientCapabilities":{}}}}
```

请求永远是自包含的：任意一台服务端实例都能处理任意一个请求，不需要粘性会话，也不需要共享的会话存储，放在普通轮询负载均衡后面就能跑。

需要提前了解服务端能力时，可以调用 `server/discover`——它不强制，只是一个可选的发现入口（在 stdio 上还兼做旧版本探测），返回支持版本、能力清单，以及可选的 `instructions`（给模型的自然语言用法说明）。旧服务不需要改动：新客户端遇到旧版服务时会退回 `initialize` 握手。

### 顺带变简单的运维

- **请求头路由**：`Mcp-Method`、`Mcp-Name` 必带，网关、WAF、限流器不用解析 JSON 体就能路由和计量；
- **列表可缓存**：`tools/list`、`resources/list` 等返回 `ttlMs` 和 `cacheScope`，客户端可以放心缓存工具目录；
- **确定性排序**：工具列表要求顺序稳定，对 LLM 的 prompt cache 命中率友好。

### 需要一个「补偿」：显式状态

服务端如果确实要跨调用保存状态（购物车、浏览器上下文、事务），做法是**发一个显式 handle**，让模型把它当普通参数带到下一次调用：

```text
create_basket() → {"basket_id": "bsk_a1b2c3"}
add_item(basket_id="bsk_a1b2c3", sku="...")
```

状态从传输层挪到了模型可见的参数里：模型能看懂自己在维护什么，服务端也不再依赖连接关联。代价是每个 handle 都要考虑授权校验、生命周期，以及过期后的报错方式。

## 二、MRTR：无状态协议里，服务端也能「回头问」

MRTR（Multi Round-Trip Requests，多轮往返请求）是这版引入的新交互模式。

场景很常见：一次工具调用执行到一半，需要用户确认或补充信息。旧版的做法是服务端主动发一个 `sampling/createMessage` 或 `elicitation/create` 请求——这要求双向流一直开着，和有状态架构绑死。

新版的流程是「一问一答，重试完成」：

1. 客户端发 `tools/call`；
2. 服务端发现信息不够，返回 `resultType: "input_required"` 和一张 `inputRequests` 清单；
3. 客户端收集好信息，**重试**原来的请求，附上 `inputResponses`；
4. 服务端凭这些信息把请求做完。

```json
// 第一次响应：请求用户确认
{"jsonrpc":"2.0","id":1,"result":{
  "resultType":"input_required",
  "inputRequests":{"confirm":{"method":"elicitation/create","params":{
    "mode":"form","message":"将删除 38 条记录，确认吗？",
    "requestedSchema":{"type":"object",
      "properties":{"ok":{"type":"boolean"}},"required":["ok"]}}}},
  "requestState":"<服务端生成的不透明状态>"}}

// 重试：带回复（注意 id 不同，这是两个独立请求）
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{
  "name":"delete_records","arguments":{},
  "inputResponses":{"confirm":{"action":"accept","content":{"ok":true}}},
  "requestState":"<原样带回>"}}
```

两个安全点值得记住：

- `requestState` 对客户端不透明，必须原样带回；服务端要把它当**不可信输入**，做完整性保护（HMAC/AEAD）并校验，否则客户端可以篡改状态来绕过授权；
- 建议在状态里绑定调用者、加短 TTL、绑定原始请求，防止重放和跨请求复用。

对 agent 的意义很直接：**工具可以在动手前做人类确认**。Supabase 的 MCP server 就是这么用的——创建项目之前先确认费用，执行删除查询之前先确认影响。

## 三、扩展体系：Tasks、Apps、Skills 与认证

这版把「扩展」做成了正式机制：可选、双方显式声明、在 `server/discover` 的能力里协商，一方不支持时优雅降级。扩展标识符形如 `io.modelcontextprotocol/tasks`（第三方用自家域名作前缀）。目前官方已有这几类：

### Tasks：长任务不再阻塞

工具调用不可能都秒回。Tasks 扩展让服务端对耗时操作返回一个持久化任务句柄，而不是挂起连接：

| 方法 | 作用 |
| --- | --- |
| 创建 | 请求的响应变成 `resultType: "task"`，带 `taskId`、`ttlMs`、建议轮询间隔 |
| `tasks/get` | 轮询任务状态 |
| `tasks/update` | 任务处于 `input_required` 时提交用户输入 |
| `tasks/cancel` | 请求取消（协作式，服务端可以不接受） |
| `notifications/tasks` | 可选的状态推送，走 `subscriptions/listen` |

任务有 `working`、`input_required`、`completed`、`failed`、`cancelled` 五种状态；完成后，`result` 字段里就是要返回的最终结果。因为 taskId 是持久句柄，客户端断线重启后还能接着轮询——CI 流水线、批量处理、人工审批、包装外部异步 job 都适合用它。

### Apps：工具也能返回 UI

MCP Apps 允许工具在描述里声明一个 `ui://` 资源，宿主把它渲染成对话内嵌的沙箱 iframe：图表、表单、PDF/视频预览、实时仪表盘都行。App 和宿主之间通过 postMessage 通信，可以反过来调 MCP 工具、接收宿主推送的新数据，也能把动作交还给宿主去调用用户已连接的其他能力。因为跑在沙箱里，宿主不需要完全信任 server 作者就可以渲染第三方 UI。Claude、ChatGPT、VS Code Copilot、Cursor、Goose 等客户端已支持。

### Skills：把流程知识放进 server

Skills 扩展复用上一篇文章讲过的 Agent Skills 规范：server 通过 `skills/list` 暴露技能目录（name/description + 文件清单），客户端按需用 `resources/read` 读取 `SKILL.md` 和附属文件；宿主会校验字节数和 SHA-256、拿到用户批准后才加载。

它解决了一个实际问题：**能力描述和能力本身一起分发**。以前工具多了，要在客户端塞一个 skill 讲「怎么用这些工具」；现在 server 可以自己提供，渐进式披露的三层结构（目录 → 指令 → 资源）在 MCP 上同样成立。

### 认证扩展

面向机器和企业的两个官方扩展：OAuth Client Credentials（服务间认证，无需交互式登录）、Enterprise-Managed Authorization（企业 IdP 集中授权）。核心协议本身也做了授权加固，见下文。

## 四、其他值得知道的修订

- **`subscriptions/listen`**：统一的长连接通知流，替代 `resources/subscribe` 和 HTTP GET 端点；客户端按类型订阅（工具/提示词/资源列表变化、指定资源更新）。
- **日志改为按请求控制**：`logLevel` 放进 `_meta`；删掉 `logging/setLevel` 和 `ping`。
- **错误码对齐 JSON-RPC**：资源不存在从 `-32002` 改为 `-32602`；`-32020`–`-32099` 划归规范保留。
- **`x-mcp-header`**：工具参数可以镜像成 HTTP 头（如 `Mcp-Param-Region`）供网关路由；敏感参数不要标。
- **可观测性**：OpenTelemetry 的 trace 上下文（`traceparent`、`baggage`）约定写进 `_meta`。

## 五、弃用清单

这版启用了正式的功能生命周期：弃用后至少保留 12 个月，期间功能照常可用。

| 弃用项 | 建议替代 |
| --- | --- |
| Roots | 工具参数、资源 URI、服务端配置 |
| Sampling | 服务端直接集成 LLM 供应商 API |
| Logging | stderr（stdio）或 OpenTelemetry |
| HTTP+SSE 传输 | Streamable HTTP |
| OAuth DCR（动态客户端注册） | Client ID Metadata Documents |

其中 Sampling 的弃用最能说明生态的成熟：协议不再鼓励「server 借 client 的模型」这种反直觉的代理关系，需要模型的 server 直接对接模型 API 即可。

## 六、MCP 配合 agent 能做什么

一句话定位：MCP 是 agent 的**接口层**——host 负责界面、权限和编排，server 负责提供能力，模型通过 client 使用它们。把上面所有能力放在一起，可以整理成一张清单：

| 需求 | 用到的原语/扩展 | 典型场景 |
| --- | --- | --- |
| 拿数据、给上下文 | resources | 文件、日志、数据库记录、文档 |
| 执行动作 | tools | 调 API、写数据库、跑命令 |
| 中途确认、补充信息 | MRTR / elicitation | 破坏性操作确认、费用确认、收集参数 |
| 跑长任务 | Tasks | CI、批处理、人工审批、外部 job |
| 富交互界面 | Apps | 图表、表单、预览、实时面板 |
| 流程与领域知识 | Skills | SOP、审查清单、团队约定 |
| 感知变化与进度 | subscriptions / progress | 列表变更、资源更新、长操作进度 |

一次完整的 agent 工具调用，大致是这个顺序：

1. 客户端调用 `server/discover`（可选）了解服务端能力；
2. `tools/list` 拉取工具目录并缓存；
3. 模型根据用户意图选择工具，客户端发 `tools/call`；
4. 需要确认或补充参数 → `input_required` → 用户回复 → 重试；
5. 操作耗时较长 → 返回 task 句柄 → 轮询 `tasks/get`，中途可 `tasks/update`；
6. 拿到结果；期间如有列表或资源变化，通过 `subscriptions/listen` 收到通知。

两个容易踩的工程点：

- **错误分两层**。请求本身的问题（未知工具、参数不符合 schema）走 JSON-RPC 协议错误；业务执行失败（日期格式不对、余额不足）放在工具结果里用 `isError: true` 返回，让模型有机会自我纠正。
- **状态靠 handle 传递**。无状态架构下，跨调用的上下文要设计成模型可见、可传递、有生命周期的参数，而不是藏在服务端内存里。

## 七、上手验证

用 Python SDK v2（`mcp` 2.3.0）写一个最小 server，先 `uv add "mcp[cli]"` 安装：

```python
from mcp.server import MCPServer

mcp = MCPServer("greeting-server")


@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b


@mcp.resource("greeting://{name}")
def greeting(name: str) -> str:
    return f"Hello, {name}!"


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

SDK 支持内存直连，测试不需要起子进程：

```python
from mcp.client import Client

async def main():
    async with Client(mcp) as client:          # 同一个进程
        await client.call_tool("add", {"a": 1, "b": 2})
```

本机运行的真实返回：

```text
content=[TextContent(text='3')], structured_content={'result': 3},
is_error=False, result_type='complete'
```

返回里已经带上新协议的 `result_type`（资源读取还会带 `ttl_ms`、`cache_scope`），说明 SDK 端到端支持 `2026-07-28`。TypeScript v2 的 server 包是 `@modelcontextprotocol/server`；四个 Tier 1 SDK（TypeScript、Python、Go、C#）都已支持新版本，Rust SDK 处于 beta。

## 八、生态与下一步

客户端方面，Claude、ChatGPT、VS Code GitHub Copilot、Cursor、Goose、Postman 等都已接入 MCP，部分已支持 Apps/Tasks/Skills 扩展（官方维护一份客户端支持矩阵）。

官方 8 月更新的 roadmap 列了五个方向：

1. **Agent 消息原语**：webhooks/channels 等服务端主动事件，让客户端不再只能轮询；Tasks 继续成熟并争取进核心；
2. **HTTP 原生传输统一**：把本地 stdio 场景也统一到 Streamable HTTP；
3. **Agent 身份与企业安全**：DPoP、Workload Identity Federation、token exchange；
4. **原语改进**：统一工具结果契约；渐进式发现（大工具目录按需展开，别让模型一开始就背一百个工具定义）；
5. **SDK 开发者体验**。

## 总结

- **2026-07-28 是结构性改版**：协议核心无状态化，`initialize` 握手和 `Mcp-Session-Id` 都移除了，每个请求自描述，服务端可以水平扩容。
- **MRTR 解决了无状态下的交互**：服务端返回 `input_required`，客户端带 `inputResponses` 重试；`requestState` 必须做完整性保护。
- **扩展体系落地**：Tasks（长任务）、Apps（内联 UI）、Skills（流程知识）、认证扩展各有清晰分工，双方 opt-in。
- **一批旧功能进入 12 个月弃用期**：Roots、Sampling、Logging、HTTP+SSE、OAuth DCR。
- **对 agent 来说**，MCP 现在不止是「调用工具」：数据、动作、确认、长任务、UI、知识、通知都有对应的标准原语。
- **对工程的要求**：跨调用状态用显式 handle，工具错误留给模型自纠，服务端日志走 stderr/OTel，敏感操作保持人类在环。

最后补一句和上一篇文章的呼应：**MCP 管「能调用什么」，Skills 管「该怎么干活」，一个负责接口、一个负责流程**。现在 Skills 也能通过 MCP 分发，这套组合基本补齐了 agent 的外置能力层。

延伸阅读：官方[规范](https://modelcontextprotocol.io/specification/2026-07-28)、[变更清单](https://modelcontextprotocol.io/specification/2026-07-28/changelog)、[发布说明](https://blog.modelcontextprotocol.io/posts/2026-07-28/) 和[扩展文档](https://modelcontextprotocol.io/extensions/overview)。
