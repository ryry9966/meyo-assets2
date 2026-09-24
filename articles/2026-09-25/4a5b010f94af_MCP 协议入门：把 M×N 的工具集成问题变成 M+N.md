---
title: MCP 协议入门：把 M×N 的工具集成问题变成 M+N
feedId: 38841
source: 综合讨论
publishedAt: 2026-09-25
---

# MCP 协议入门：把 M×N 的工具集成问题变成 M+N

## 背景

做 Agent 的人都遇到过同一件事：模型不缺智力，缺"手"。读本地文件、查数据库、调内部 API、操作浏览器，全靠外部工具补齐。在 MCP 出现之前，这些接入全靠各家自定义的插件格式。2024 年底 Anthropic 开源了 Model Context Protocol（MCP），一年多下来，它基本成了 Agent 接工具的事实标准：官方 SDK 覆盖 Python/TypeScript，主流 IDE 和 OpenClaw 这类 Agent 运行时都内置了客户端支持。

## 它到底解决了什么问题

MCP 之前，工具接入是典型的 M×N 问题：M 个 Agent 框架 × N 个工具，每对组合都要写一份胶水代码。你的文件读取插件在框架 A 是一种写法，到框架 B 要重写一遍；换个模型，函数定义格式又不一样。

MCP 把 M×N 压成 M+N：工具方只需实现一次 MCP Server，任何支持协议的 Host 都能直接用。协议本身很朴素——基于 JSON-RPC 2.0，定义了三个核心原语：

- **Tools**：模型可调用的动作，如"执行 SQL"、"发消息"；
- **Resources**：应用可控的上下文数据，如文件内容、配置；
- **Prompts**：用户可选的模板。

加上统一的工具发现（`tools/list`）、调用（`tools/call`）和 JSON Schema 参数描述，"一个工具长什么样"第一次有了通用答案。

## 最小上手路径

以 Python SDK（FastMCP）为例，三步：

**1. 写一个 Server，暴露一个工具：**

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo-tools")

@mcp.tool()
def get_table_schema(table: str) -> str:
    """返回指定数据表的字段结构，构造查询前先调用确认字段名。"""
    return DB_SCHEMA.get(table, "table not found")

mcp.run(transport="stdio")
```

**2. 在 Host 侧注册**：stdio 模式下就是在配置里声明一条启动命令，Host 拉起该子进程并维持会话。

**3. 按顺序验证**：先确认 `tools/list` 能枚举出工具，再手动 `tools/call` 一次，最后才让模型自主调用。

顺序别反。跳过前两步直接让 Agent 跑，出问题时分不清是协议层还是模型层的锅。

## 踩坑点

- **stdio 模式下别往 stdout 打日志**。stdout 走的是 JSON-RPC 帧，一行 print 就能让整个会话挂掉。日志一律走 stderr 或文件。
- **工具描述即提示词**。模型选不选、怎么选工具，几乎取决于 name、description 和参数注释。`do_stuff` 这种命名和含糊的 docstring，是"模型不会用工具"的头号根因。
- **工具数量吃上下文**。十几个 Server 全挂上，光工具定义就占几 KB，直接压缩可用推理空间。按任务域挂载，用不到的先关。
- **第三方 Server 等于供应链**。MCP Server 以你的权限运行，工具描述还可能被注入恶意指令（tool poisoning）。来源不明的 Server 不要接生产。
- **协议在演进**。2025 年的规范用 Streamable HTTP 取代了旧的 HTTP+SSE 传输，并补充了 OAuth 授权。客户端与服务端 SDK 版本差太多会有兼容问题，升级前看 changelog。

## 可复用的建议

- 一个 Server 只管一个领域；工具保持细粒度、幂等，别做"一键完成一切"的大工具。
- 返回值要短、结构化、成败可判断；报错信息是写给模型看的，要让它知道下一步该怎么办。
- 把工具描述当 prompt 来维护，和代码走同样的 review 流程。
- 个人本地自动化，stdio 足够；要团队共享或跨机访问，再上 HTTP 传输加鉴权。

## 总结

MCP 没有黑魔法，它就是一层管道标准：把"模型怎么发现工具、怎么调用工具、怎么拿到上下文"统一了。它的价值不在单机 demo——一个脚本里直接函数调用更快——而在于生态收敛之后，你写的工具能被任何 MCP 客户端复用，接入新 Agent 的边际成本趋近于零。如果你在维护多个 Agent 或多个内部系统，值得现在就把工具层 MCP 化；如果只是一个一次性的自动化脚本，不必为了用而用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/ce840775e45e3c45.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/0aa4221f273438e1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/27cc83217941d826.png)

