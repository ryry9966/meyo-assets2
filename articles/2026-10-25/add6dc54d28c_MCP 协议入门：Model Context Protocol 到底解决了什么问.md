---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42559
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

做 Agent 自动化的人对"接工具"的碎活都不陌生：每个框架有自己的插件格式，接一个内部 API 要写 schema、写描述、写鉴权胶水代码，换个宿主全部重来一遍。

这就是经典的 M×N 问题：M 个宿主 × N 个工具，最坏要写 M×N 份适配代码。MCP（Model Context Protocol）的思路是把乘法变加法：工具方实现一次 MCP Server，任何支持 MCP 的宿主都能直接用。

常有人说 MCP 是"AI 的 USB-C"，这个类比有点营销味，但方向对：它不规定工具能干什么，只规定双方怎么对话。

## 问题

MCP 具体标准化了三件事：

1. **发现机制**：宿主连上 Server 后自动列出可用的 tools、resources、prompts，无需手工登记；
2. **调用契约**：统一的 JSON-RPC 消息与 JSON Schema，宿主负责转成模型侧的 function calling 格式；
3. **传输层**：本地 stdio、远程 Streamable HTTP，含协议版本协商。

没解决的也要说清：不保证描述质量（模型选不选得对全看 description）、不统一鉴权（远程 Server 各家实现差异大）、更不解决工具本身设计烂的问题。

## 做法与步骤

用 Python 官方 SDK 走一条最小路径：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("ops-tools")

@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查询订单状态，返回 JSON 字符串；找不到返回空对象。"""
    return get_order(order_id)

mcp.run()  # 默认 stdio 传输
```

1. 在 OpenClaw 或其他宿主的配置里注册这个 Server（本地就是一条启动命令加参数）；
2. 连接后先调 `tools/list`，核对工具名、参数 schema、描述——这一步比急着跑通更重要；
3. 用真实输入调 `tools/call`，确认返回结构和错误路径（超时、异常时返回什么）；
4. 都没问题再接入 Agent 主流程。

## 踩坑点

- **描述即提示词**：模型按 description 选工具。多个语义相近的工具共存时，含糊描述几乎必然选错。写清楚干什么、入参含义、失败时返回什么。
- **工具数量膨胀**：一个 Server 挂几十个工具，上下文占掉一大截，选择准确率肉眼可见地下滑。按场景拆分 Server，宿主侧按需启用。
- **非幂等工具遇自动重试会出事故**：Agent 层的重试逻辑碰上"发消息、下单"类工具会重复执行。写操作要么做幂等键，要么在工具内部加确认/干跑机制。
- **stdio 的环境坑**：Windows 下编码、解释器路径问题会让连接静默失败。先在终端手工跑一遍启动命令，能起来再谈集成。
- **远程部署别裸奔**：HTTP Server 暴露即公网可达，认证和限流要在网关或 Server 层自己做，SDK 默认不带。

## 可复用建议

- 内部系统只写一次 Server、多宿主复用，这是 MCP 最划算的回报点；
- 粒度宁小勿大，一个工具干一件事，组合逻辑交给模型或编排层；
- 每次调用落日志（入参、出参、耗时、是否重试），排查 Agent 行为时这是唯一可靠证据；
- 改工具签名时保持向后兼容，或改名加版本号——宿主缓存的 schema 不会立刻更新。

## 总结

MCP 解决的是工程问题，不是智能问题：它把工具接入从一次性定制变成了标准化契约。如果你已经在多个宿主之间重复写适配代码，值得花一个下午跑通最小示例；如果只有一个宿主、三五个工具，原生插件机制够用，不必为了 MCP 而 MCP。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/e58c7bad97995e1c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/4c1494b335db5f35.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/14b3433cb1ddebb3.png)

