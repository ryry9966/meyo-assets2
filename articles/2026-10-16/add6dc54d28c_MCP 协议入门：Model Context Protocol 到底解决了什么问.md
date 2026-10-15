---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 41647
source: 综合讨论
publishedAt: 2026-10-16
---

## 背景

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），一年多下来基本成了 Agent 接工具的事实标准。OpenClaw 社区里的很多自动化场景——查库、调内部 API、批量操作文件——都在往 MCP 上迁移。但不少人只是照抄配置文件，并不清楚这个协议替我们省掉了什么。这篇从工程角度把问题讲清楚。

## 它到底解决什么问题

在 MCP 之前，模型接工具是纯手工作坊：每个 Agent 框架有自己的 function calling 格式，每个工具要为每个框架单独写一遍适配。M 个模型 × N 个工具，就是 M×N 份胶水代码。而且工具能力没有发现机制——换一个客户端宿主，工具清单、参数说明全要重写。

MCP 的答案是把 **M×N 压成 M+N**：工具方只需实现一次 MCP server，任何支持 MCP 的客户端都能动态发现并调用它。协议基于 JSON-RPC 2.0，核心只有三类原语：

- **tools**：模型可调用的动作（查订单、跑脚本）
- **resources**：可读取的上下文数据（文件、表结构）
- **prompts**：可复用的提示模板

## 做法：跑通一个最小闭环

1. **分清角色**。Host（如 OpenClaw）内嵌 MCP client；每个 client 与一个 MCP server 保持 1:1 连接；server 才是真正包着工具逻辑的进程。
2. **理解生命周期**。连接建立时先 `initialize` 握手、协商双方能力；之后 client 用 `tools/list` 拉取工具清单，用 `tools/call` 发起调用。工具发现是动态的——server 加了新工具，客户端不用改配置。
3. **写一个最小 server**。Python SDK 几行就能跑：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("ops-tools")

@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查询订单状态，返回状态、金额和最近一次物流事件。"""
    ...

mcp.run(transport="stdio")
```

装饰器把函数暴露成 tool，**docstring 就是模型看到的工具说明**。stdio 传输下，client 直接把它当子进程拉起，stdin/stdout 走 JSON-RPC。
4. **先验证再接入**。用官方 MCP Inspector 跑一遍 `tools/list` 和 `tools/call`，确认描述和返回结构没问题，再挂到 Agent 上。

## 踩坑记录

- **stdio server 千万别 print 到 stdout**。stdout 是协议通道，一句调试日志就能让连接解析失败。日志一律走 stderr。
- **注意传输协议版本**。网上大量教程基于 SSE，2025-03-26 版规范已改为 Streamable HTTP，老配置不一定能跑。SDK 版本和规范版本要对照着看。
- **工具描述含糊，模型就会乱调用**。描述是写给模型看的 prompt，不是给人的注释：说清楚做什么、参数格式、什么情况不该调用。
- **工具列表别贪多**。一次挂几十个工具，选择准确率明显下降；删数据、外发请求这类危险操作，务必在 server 侧加确认或白名单。
- **早期版本没有内建鉴权**，HTTP 形式的 server 不要裸奔到公网。

## 可复用的建议

- **按领域拆 server**：文件系统一个、数据库一个、内部 API 一个，别做全能单体。
- **返回值精简、结构化**。把整张表灌进上下文既贵又拉低调用质量，给摘要加关键字段即可。
- **长任务用 progress 通知**，避免客户端超时重试导致重复执行。
- **锁定 SDK 版本**，规范还在快速演进（2025-06 版加入了 elicitation 和更完整的 auth），升级前先用 Inspector 做一轮回归。

## 总结

MCP 没有让模型更聪明，它只是把"接工具"这件事标准化了。真正的价值在于把集成成本从 M×N 降到 M+N，让工具逻辑回归工具方、编排回归 Agent。对 OpenClaw 用户来说，把一个你重复封装过多次的内部接口改写成 MCP server，通常一个下午就够——回报是之后所有 MCP 客户端都能免费复用这份工作。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/ac8b5b4b02477813.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/e9aa84144f64c41e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/27f3c7e50f2325b5.png)

