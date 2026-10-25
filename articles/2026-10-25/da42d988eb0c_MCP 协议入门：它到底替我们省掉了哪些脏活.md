---
title: MCP 协议入门：它到底替我们省掉了哪些脏活
feedId: 42582
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

Agent 要真正干活，绕不开两件事：给模型供上下文，让它能调外部工具。在 MCP 出现之前，这两件事没有统一做法——每个宿主（IDE、桌面助手、自研 Agent）想接一个数据源或工具，都得手写一套私有适配。三个宿主、五个工具，就是十五份胶水代码，且协议、鉴权、Schema 各不相同。

2024 年底 Anthropic 把 Model Context Protocol（MCP）开源出来，基于 JSON-RPC 2.0，定位很克制：把「Agent ↔ 工具/数据源」这条接口标准化，类似外设之于 USB-C。

## 它解决的具体问题

- **M×N 变 M+N**。工具方写一次 MCP Server，宿主方实现一次 Client，彼此即插即用，不用再两两适配。
- **上下文供给标准化**。协议定义三类原语：`tools`（模型可调用）、`resources`（宿主可读的上下文）、`prompts`（可复用模板）。支持哪些能力，在 `initialize` 握手时协商清楚。
- **传输统一**。本地进程走 stdio，远程服务走 Streamable HTTP，不再各造通信轮子。

## 最小上手路径

1. 装官方 SDK（Python 的 `mcp`，或 TS 的 `@modelcontextprotocol/sdk`）。
2. 先写一个 stdio Server，FastMCP 十行以内：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("repo-tools")

@mcp.tool()
def grep_repo(pattern: str, path: str = ".") -> str:
    """按正则搜索仓库代码，返回匹配行与所在文件路径。"""
    ...
```

3. 挂到宿主验证：桌面客户端配置里注册一条 command，或在自己的 Agent 循环里起一个 Client。
4. 需要远程共享时，再把 Server 换成 Streamable HTTP 部署，鉴权按规范走 OAuth 2.1，别自己发明。

## 踩坑点

- **stdout 是协议通道**。stdio 模式下往 stdout `print` 调试信息，会直接打断 JSON-RPC 流，Server 表现为「静默失联」。日志一律走 stderr 或文件。
- **工具描述就是给模型看的文档**。描述含糊、没写清参数格式和边界条件，模型必然乱调。写描述的时间应该多于写函数体。
- **Schema 别贪宽**。参数全用宽泛 string，模型会自己「编」格式出来；类型收窄、加枚举和默认值，命中率明显提升。
- **返回值别灌大块内容**。一次性返回整文件、整表，上下文窗口立刻被打爆。返回句柄 + 分页读取，是更稳的模式。
- **多 Server 工具重名**。宿主侧要做前缀或路由策略，否则模型选错工具的概率不低。
- **版本协商失败**，多半是 Client 和 Server 的 SDK 版本差太远，先对齐协议版本再排查别的。

## 可复用建议

- Server 只做薄适配层，鉴权和业务逻辑留在自己的服务里，协议层保持「傻」一点。
- 工具设计面向模型人体工学：小而正交、命名直白、描述里带一个反例。
- 读多写少的上下文用 resources，动作类用 tools；危险动作（删数据、发消息）在宿主层加人工确认，别只依赖模型自觉。
- 每次调用记 trace 日志，出问题时这是唯一能救你的东西。

## 总结

MCP 不负责让你的工具变好用，它负责让工具可插拔。它把接口格式、能力协商、上下文供给、传输方式这几件重复造轮子的事标准化了，收益是集成的边际成本趋近于常数。真正的难点仍在工具设计和上下文卫生——这两件事，协议替不了你。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/ff5aa39fbf249c0b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/bb59486ce6548826.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/36db87df0f1e4dc6.png)

