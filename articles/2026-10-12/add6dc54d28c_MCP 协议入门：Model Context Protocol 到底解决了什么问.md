---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 41265
source: 综合讨论
publishedAt: 2026-10-12
---

## 背景：M×N 集成困境

做 Agent 开发绕不开一个现实：模型只会生成文本，要干活就得接外部能力——查数据库、调 API、读文件、发消息。这件事长期以来没有统一做法：每个 Agent 框架有自己的插件格式，每个工具方提供自己的 SDK，接一个新工具就要重写一遍胶水代码、鉴权和参数对齐。

维护过两个以上 Agent 的人都体会过这个乘法：M 个应用 × N 个工具 = M×N 份集成代码。

## 问题：MCP 到底标准化了什么

MCP（Model Context Protocol）由 Anthropic 于 2024 年底开源，本质是把「模型应用如何连接工具与数据」这件事收敛成一个开放协议。核心三点：

- **架构**：Host-Client-Server 三层。Agent（Host）内嵌 MCP Client，通过 JSON-RPC 与 MCP Server 通信。
- **原语**：Server 只暴露三类能力——Tools（模型可调用的动作）、Resources（应用可控的上下文数据）、Prompts（用户可触发的模板）。
- **传输**：本地 stdio、远程 Streamable HTTP 两条标准路径。

关键认知：MCP 解决的不是「模型怎么变聪明」，而是「能力如何被描述、被发现、被调用」这个接口层问题。工具实现一次，任何支持 MCP 的 Host 都能挂载，M×N 降成 M+N。同时也要清楚它没解决什么：鉴权体系、工具质量、安全审计依然是接入方自己的事。

## 做法：最小可用 Server

以 Python SDK 为例，十几行就能跑：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("ops-tools")

@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查询订单状态，返回 JSON 字符串。"""
    return db.get_order(order_id)

mcp.run()  # stdio 模式
```

三步接入：① 写 Server，用装饰器声明工具，函数签名和 docstring 就是模型看到的接口；② 在 Host 的 MCP 配置里声明 command/args（stdio）或 url（HTTP）；③ 先用官方 Inspector 调通，再接入 OpenClaw 这类 Agent，观察实际工具调用日志。

## 踩坑点

- **stdio 启动失败最常见**：`command not found` 多半是 PATH 问题——GUI 应用拉起的子进程不继承你 shell 的环境变量，`npx`/`uvx` 建议写绝对路径。
- **工具描述质量决定调用质量**：模型靠 description 选工具。写得含糊，要么不调用，要么乱传参。docstring 要按 API 文档的标准写，不是按注释的标准。
- **工具列表膨胀吃上下文**：一个 Server 挂 50 个工具，光 schema 就占几千 token。按场景拆 Server，按需启用。
- **长耗时工具要处理超时**：默认超时往往只有几十秒，慢查询要么加进度通知，要么拆成「提交任务 + 查询结果」两个工具。
- **不要盲信工具返回**：工具结果会拼进上下文，不可信数据源可能借 prompt injection 操纵模型。写操作做确认，外部数据做过滤。

## 可复用建议

- 一个 Server 只管一个领域（数据库、浏览器、文件系统分开），别做全能工具箱。
- 工具设计小而正交，参数用 Pydantic/Zod 严格校验，宁可报错也别让模型猜。
- Server 每次变更跑一遍回归：工具列表、参数 schema、异常格式都可能影响下游 Agent。
- 记录每次调用的入参、耗时、错误——这是排查 Agent 行为的唯一可靠依据。

## 总结

MCP 的价值不在技术多先进——JSON-RPC 和 schema 描述都不新——而在于把生态收敛到一个中立接口上。判断标准很简单：能力需要在多个 Agent 间复用，或想接入现成工具生态，就值得用 MCP 包一层；只是单应用内部调用，直接函数注册反而更省事。协议是手段，工具链路的可观测与可控才是自动化的底线。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/360ed0a8d692d6a2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/d9ae1867deb635e4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-12/5ed3e967d94aaba8.png)

