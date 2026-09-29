---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 39727
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

Agent 真正落地时，瓶颈往往不在模型本身，而在"模型怎么碰到外部世界"。要让 Agent 查数据库、调内部 API、操作日历，传统做法是给每个 Agent 框架各写一套 function calling 胶水代码：工具定义写一遍、鉴权接一遍、错误处理再来一遍。换一个框架，这些工作基本重做。

2024 年底 Anthropic 把 MCP（Model Context Protocol）开源出来，思路很直接：把"模型 ↔ 工具/数据"这一层的接口标准化。它是 JSON-RPC 之上的一套协议约定，不是新框架，也不绑定某个厂商。

## 它到底解决什么问题

核心是 M×N 集成问题：M 个模型/Agent 框架 × N 个工具，两两组合都要写适配，复杂度是乘积。MCP 把它压成 M+N——工具方实现一次 MCP Server，任何支持协议的 Host 都能接入；Agent 侧实现一次 MCP Client，能挂上所有 Server。

协议里真正被标准化的东西是三类原语：

- **Tools**：模型可以主动调用的动作（发请求、写文件）；
- **Resources**：可以被读进上下文的数据（文件、配置、查询结果）；
- **Prompts**：预置的提示模板。

再加上统一的发现机制（Host 启动时向 Server 拉取工具列表和 JSON Schema）、两种传输方式（本地 stdio、远程 Streamable HTTP），插件生态第一次有了"插上就能用"的可能。

## 最小可跑的做法

用官方 Python SDK 写一个只有两个工具的 Server，十几行：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("team-tools")

@mcp.tool()
def get_ticket(id: str) -> str:
    """按工单 ID 查询状态，返回标题、负责人、当前阶段。"""
    return query_db(id)

@mcp.tool()
def add_note(id: str, content: str) -> str:
    """向指定工单追加一条备注。"""
    return append_db(id, content)

mcp.run(transport="stdio")
```

三个关键点：

1. 工具描述（docstring）会原样进入模型上下文，等于写给模型看的 API 文档，别敷衍。
2. 参数类型注解会被转成 JSON Schema，Host 靠它做参数校验和补全。
3. 本地场景用 stdio 最省事：Host 以子进程方式拉起 Server，生命周期跟着会话走。

跑起来后在支持 MCP 的 Host（Claude Desktop、OpenClaw 的 Agent 运行时等）里注册这个 Server，先确认工具列表能被发现，再让 Agent 实际调用一次并看日志。

## 踩坑点

- **工具越多，上下文越脏**。每个工具定义都占 token，挂 50 个工具后模型选错工具的概率明显上升。按领域拆 Server，按需启用。
- **工具描述是行为的一部分**。描述含糊，模型就会猜参数、编结果。写清输入、输出、副作用。
- **安全边界常被低估**。stdio 模式下 Server 以你的用户权限运行，Agent 能调它就等于能做它做的事。危险操作要在 Server 内部加确认或白名单，不能指望模型自觉。
- **Schema 要严格**。可选参数给默认值，枚举就写 enum，别搞"字符串里塞 JSON"这种模糊结构。
- **长任务别阻塞**。超过几十秒的操作，考虑返回任务 ID 加一个查询工具，而不是让一次调用挂到超时。

## 可复用的建议

- 一个 Server 只管一个领域（工单、日历、文档），不要造"万能网关"。
- 工具尽量幂等、粒度小，把"查列表"和"查详情"拆开，比返回一大坨合并数据好。
- 所有工具调用留日志：入参、出参摘要、耗时、调用方。排查 Agent 异常行为时，这是唯一的证据链。
- 先在本地 stdio 跑通全链路，再迁到远程 HTTP 加鉴权；顺序反过来会同时 debug 两层问题。

## 总结

MCP 没有让 Agent 变聪明，它做的是更朴素的事：把接入成本从 M×N 压到 M+N，让工具和 Agent 各自只实现一次。对做自动化的我们来说，价值在于工具终于可以按"服务"来维护，而不是按"某个框架的插件"。建议路径：先写一个只有两三个工具的小 Server 跑通全链路，再谈生态和复用。

---

