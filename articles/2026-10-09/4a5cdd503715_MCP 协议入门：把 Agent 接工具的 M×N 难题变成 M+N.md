---
title: MCP 协议入门：把 Agent 接工具的 M×N 难题变成 M+N
feedId: 40968
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

Agent 的能力边界，很大程度取决于它能碰到哪些工具和数据。模型本身只会生成文本，真正“能干活”靠的是外接函数调用：查数据库、读文件、发消息、操作浏览器。

在 MCP 出现之前，这件事没有统一做法。每个 Agent 框架都有自己的插件格式，每个工具都要为不同框架写一份适配。M 个应用、N 个工具，就要写 M×N 个连接器——工具方追不动，Agent 开发者各自重复造轮子。

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），把“模型应用 ↔ 工具”这一层抽成标准协议，消息层基于 JSON-RPC 2.0。现在 Claude Desktop、Cursor、OpenClaw 等大量 Host 都能直接挂 MCP server。

## 它具体解决了什么

MCP 把 M×N 收敛成 M+N：工具方写一次 server，任何实现客户端的 Agent 都能用；Agent 侧实现一次客户端，就能接入所有 server。

三个角色要分清：

- **Host**：Agent 本体（如 OpenClaw），内嵌 MCP Client；
- **Server**：暴露能力的轻量进程，本地子进程走 stdio，远程服务走 Streamable HTTP；
- **能力三件套**：Tools（模型可调用的函数）、Resources（可读取的数据）、Prompts（可复用模板）。

连接建立时先握手协商能力，client 拉取 `tools/list` 拿到工具清单和参数 schema 注入模型上下文；模型决定调用时走 `tools/call`，结果回填给模型。工具发现、参数校验、结果回传从此有了统一语义。

## 在 OpenClaw 里跑通最小流程

1. 先用现成 server 验证链路：filesystem、fetch、Playwright 这类社区 server 覆盖大部分需求；
2. 在 OpenClaw 配置里登记 server：本地进程写启动命令，远程服务写 URL 和鉴权；
3. 重启会话，确认 agent 的可用工具列表里出现了新工具；
4. 自建 server 用官方 SDK 十几行即可：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("ops-tools")

@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查询订单，返回状态、金额、物流单号。"""
    return db.lookup(order_id)

mcp.run()
```

关键就一件事：**工具描述是写给模型看的**。它靠这段话决定何时调用、传什么参数。

## 踩坑点

- stdio 模式下，server 往 stdout 打日志会直接破坏协议流。日志一律走 stderr，这是新手最常见的翻车点。
- 工具数量失控会稀释上下文：描述占 token，模型选错工具的概率也上升。按需启用，一个 server 只管一个领域。
- 远程 server 的鉴权别裸奔：token 用环境变量注入，确认配置文件不会被提交进仓库。
- 规范在演进（Streamable HTTP 已取代早期 HTTP+SSE），握手失败先查两端协议版本；长耗时工具还要处理超时与进度反馈，否则 agent 侧表现为“卡死”。

## 可复用建议

- **选型判断**：能力只服务于 OpenClaw 内部逻辑，写原生插件更顺；能力是通用的（查库、发工单、控浏览器），写成 MCP server，一次投入多端复用。
- 工具设计偏粗粒度，返回结果做裁剪——模型不需要一万行原始 JSON，给它结论和关键字段。
- 接进 agent 之前，先用 MCP Inspector 单独调试 server，能提前暴露大部分问题。
- 把 MCP server 当不可信边界处理：校验参数、限制文件访问范围，别因为“自己写的”就放松。

## 总结

MCP 不是让模型变聪明的魔法，它只是一层标准化管道：把“每接一个工具都要重写一遍”的脏活，收敛成协议两端各自做一次。对 OpenClaw 用户的务实路径是：先用现成 server 跑通链路，理解 Tools / Resources / Prompts 的边界，再决定是否为自己的高频场景写一个。管道铺好了，能力才有地方接。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/59f117b3c2ce318c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/2c9d0c9936ef9018.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/4f50df99e6b966a2.png)

