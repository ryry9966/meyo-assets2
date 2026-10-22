---
title: MCP 协议入门：把 M×N 的工具接入问题压成 M+N
feedId: 42325
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

做 Agent 开发绕不开一个现实：模型本身只会生成文本，真正干活的是外部工具和数据源——查数据库、调内部 API、读文件、发消息。2024 年底 Anthropic 开源了 Model Context Protocol（MCP），一年左右时间，它基本成了 Agent 生态连接工具的事实标准，OpenClaw 的插件生态也在向它靠拢。值得把它的底层逻辑搞清楚再上手。

## 它到底解决了什么问题

MCP 之前的状态可以概括成 **M×N 问题**：M 个 Agent 应用对接 N 个工具，就要写 M×N 份胶水代码。每个工具的鉴权、参数格式、错误返回各不相同，工具作者要为每个客户端单独适配，Agent 开发者要为每个工具单独写 connector。

MCP 把 M×N 压成 **M+N**：工具方实现一次 MCP Server，Agent 方实现一次 MCP Client，两边通过统一协议见面。

协议本身不神秘，核心是三件事：

1. **传输层**：基于 JSON-RPC 2.0，本地走 stdio（子进程），远程走 Streamable HTTP。
2. **三个原语**：Server 可向 Client 暴露 Tools（可执行动作）、Resources（可读数据）、Prompts（预设模板），在 initialize 握手时协商能力。
3. **动态发现**：Agent 运行时通过 tools/list 拿到工具清单和 JSON Schema，再通过 tools/call 调用，不需要硬编码。

## 动手跑通最小链路

以 stdio 为例，三步：

1. 用官方 Python SDK 写一个 Server：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo-tools")

@mcp.tool()
def get_order(order_id: str) -> str:
    """按订单号查询订单状态"""
    return f"order {order_id}: shipped"

mcp.run()
```

2. 在 Agent 宿主的配置里注册这个 Server（启动命令、参数、环境变量）。
3. 启动宿主，确认工具被列出，跑一次真实调用。

调试用官方 MCP Inspector，能直接看握手过程、工具列表和调用返回，比在 Agent 里盲调快得多。

## 踩坑点

- **工具描述就是 prompt**。模型调用与否完全取决于 description 和参数说明，写得含糊就会调错。这不是 SDK 问题，是提示工程问题。
- **工具越多，上下文越贵**。每个 Server 的工具 schema 都占 token。挂七八个 Server、每个十几把工具，模型选工具的准确率会肉眼可见地下降。按域拆分、按需挂载。
- **stdio 生命周期**。子进程由宿主拉起，路径用绝对路径，Python 注意用对应 venv 的解释器，环境变量不会继承你的 shell。
- **区分执行错误和协议错误**。业务失败应通过 isError 返回给模型，让它自行重试或换路，而不是抛 JSON-RPC error 直接断链。
- **安全别裸奔**。第三方 Server 的返回值会直接进入模型上下文，存在注入风险。文件系统类 Server 务必收敛到最小目录，敏感操作在宿主侧加人工确认。

## 可复用建议

- 一个 Server 只管一个域，工具数量控制在个位数。
- 写 description 时假设读者完全不了解你的业务——模型确实不了解。
- 只读数据用 Resources，动作用 Tools，别把查询包装成有副作用的工具。
- 在宿主侧统一记录每次 tools/call 的入参、耗时和结果摘要，出问题时这是唯一线索。

## 总结

MCP 的价值不在技术多复杂——它只是把"模型怎么调工具"这件事标准化了，类似 USB-C。真正的工程量在工具设计本身：描述写得准、粒度切得对、权限收得住。协议解决的是"接得上"，接得好不好，仍然是你的问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/9ab9eb50e1583628.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/7684acefd1683822.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/58ddd8bdb03d72e0.png)

