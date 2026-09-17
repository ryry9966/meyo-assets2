---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38009
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

大模型应用走到 Agent 阶段，绕不开两件事：调工具、读数据。文件系统、数据库、内部 API、浏览器……每个 Agent 框架都有一套自己的接入方式。MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的一套协议，目标只有一个：把"模型怎么连工具"这件事标准化。

## 它到底解决什么问题

MCP 出现之前是典型的 M×N 问题：M 个 AI 应用要接 N 个工具/数据源，理论上有 M×N 种组合。每个客户端都自定义一遍工具格式、鉴权方式、发现机制；你给 A 框架写的插件，换到 B 框架基本要重写。

MCP 把 M×N 压成 M+N：工具方只需实现一次 MCP Server，任何支持 MCP 的客户端都能直接用。可以类比 USB-C——设备不用关心插在哪台电脑上，协议统一了接口。

## 核心概念，五分钟版

- **Host / Client**：承载模型的宿主（IDE、桌面助手、Agent 框架），内部通过 MCP Client 与 Server 通信
- **Server**：暴露能力的独立进程，消息层基于 JSON-RPC 2.0
- 三类原语：**Tools**（模型主动调用的动作）、**Resources**（可读取的数据）、**Prompts**（预设模板）。实践中大部分场景只用到 Tools
- 传输层：本地进程用 stdio，远程服务用 Streamable HTTP

## 最小可运行示例

用官方 Python SDK 写一个查天气的 Server，十行以内：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("weather")

@mcp.tool()
def get_weather(city: str) -> str:
    """查询指定城市的当前天气"""
    return f"{city}: 晴, 26°C"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

在客户端配置里注册这个 Server，重启后模型就能看到 `get_weather` 工具，对话中提到天气时自动发起调用。全程不需要改 Agent 框架代码——这就是协议层标准化带来的直接收益。

## 踩坑点

1. **stdio 模式下 print 会毁掉连接**。stdout 是协议通道，日志必须写 stderr，否则 Client 解析 JSON-RPC 直接报错。这是新人第一坑。
2. **工具描述是写给模型看的**。描述含糊，模型就会在错误时机调用、或传错参数。写 description 时假设读者是不了解你业务的大模型，而不是人类开发者。
3. **不要盲目信任 Server 返回的内容**。工具输出会进入模型上下文，被污染的数据源等于注入入口。对涉及写操作、删操作的工具，务必在 Host 层加人工确认。
4. **工具数量失控**。单个 Server 挂几十个工具，模型的选择准确率明显下降。按业务域拆分，每个 Server 保持 5~15 个工具比较稳。

## 可复用建议

- 先本地 stdio 跑通，再考虑迁远程 HTTP；多数个人自动化场景本地就够。
- 只读和写操作拆成不同工具，方便后续做权限粒度。
- 错误信息返回给模型时要"可行动"：告诉它错在哪、下一步能做什么，而不是抛一段堆栈。
- Server 本身保持无状态，会话状态交给 Host 管，扩展成本最低。

## 总结

MCP 没有黑科技，本质是用 JSON-RPC 加标准化的能力描述，把工具接入从"每家一套"变成"一套走天下"。对 Agent 开发者的价值很直接：写一次工具，处处可用。它解决的是工程问题而非智能问题——但恰恰是这类朴素的标准，决定一个生态能不能长起来。建议从"把自己业务里最烦的重复操作封装成一个 MCP Server"开始，体感最直观。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/5826c29ceaed15ed.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/ccdf78ba7b985f6d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/d690f4141b859add.png)

