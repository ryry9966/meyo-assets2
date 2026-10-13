---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 41454
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

过去两年做 Agent 开发的人大多遇到过同一个困境：模型本身能力不差，但要让它"动手"，就得为每个目标系统写一遍胶水代码。接 GitHub 要写一套 function calling 的 schema，接内部数据库再写一套，换个 Agent 框架，这些集成逻辑还得重写。这就是典型的 M×N 问题：M 个应用 × N 个工具，集成成本是乘法关系。

2024 年底 Anthropic 开源了 MCP（Model Context Protocol），目标是把 M×N 拉平成 M+N：应用方只需实现一次 MCP 客户端，工具方只需实现一次 MCP 服务端，中间用统一协议对话。它基于 JSON-RPC 2.0，目前有官方的 Python 和 TypeScript SDK，生态里的 server 数量增长很快。

## 它到底解决了什么

MCP 的核心抽象只有三类原语：

- **Tools**：模型可主动调用的动作（查数据、执行命令）；
- **Resources**：应用侧可控的上下文数据（文件、记录）；
- **Prompts**：用户侧可控的预设模板。

价值不在于"又多了一个协议"，而在于它定义了清晰边界：谁控制什么、消息怎么流转、能力如何被发现（`tools/list`、`tools/call` 等标准方法）。这让"接入一个新数据源"从写代码变成装插件。

## 上手步骤

以 Python 为例，一个最小可用的 MCP server 只要十几行：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo-tools")

@mcp.tool()
def get_issue(repo: str, number: int) -> str:
    """根据仓库和编号返回 issue 概要"""
    ...

mcp.run(transport="stdio")
```

建议按这个顺序走：

1. 用 MCP Inspector（`npx @modelcontextprotocol/inspector`）先把 server 跑通，确认工具能被列出、能被调用；
2. 再接入宿主（Claude Desktop 或你自己的 Agent 框架），本地进程用 stdio，远程服务用 Streamable HTTP；
3. 观察模型的工具选择行为，再回头修改工具描述。

## 踩坑点

- **工具描述就是 prompt。** 模型选不选、怎么调用某个工具，几乎完全取决于名称和描述。描述含糊，调用就飘，参数也容易填错。
- **stdio 模式下别往 stdout 打日志。** stdout 被 JSON-RPC 流占用，`print` 一次整条会话就断了，日志一律走 stderr。
- **工具数量要克制。** 一次挂几十个工具，上下文膨胀，选择准确率明显下降，应按场景拆分加载。
- **安全上别乐观。** MCP server 以你的凭证运行，工具返回内容会被拼进模型上下文——返回值里的注入文本可能诱导模型误操作。权限最小化，写操作加确认层。
- **版本兼容。** 早期文档里的 HTTP+SSE transport 已被 Streamable HTTP 取代，老客户端对接新 server 时先确认协议版本。

## 可复用建议

- 一个 server 只做一类事，不要造"万能工具箱"；
- 描述当 prompt 写，参数名当文档写；
- 先只读后写入，写操作永远留一层确认；
- 密钥走环境变量，不进代码和配置仓库；
- 任何 server 先过一遍 Inspector 再上生产。

## 总结

MCP 没有魔法，它做的事情是把"模型如何获取上下文和工具"这件事标准化。对写 Agent 的人来说，它是减少重复集成的插件规范；对工具提供方来说，它是被所有兼容宿主直接使用的分发渠道。协议本身很薄，真正决定体验的，是你暴露给模型的每一个工具描述和权限设计。建议从一个小而窄的只读工具开始，把整条链路跑通再逐步扩展。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/16fed64d026e8d9e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/fa007562b828f8c1.png)

