---
title: MCP 协议入门：把 M×N 的集成难题变成 M+N
feedId: 42239
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

Agent 要真正干活，就得接触外部世界：查数据库、调内部 API、读文件、触发动作。在 MCP 出现之前，这一步基本靠各家手写胶水代码——每个 Agent 框架实现一套 tool calling，每个工具方再为每个框架单独适配。MCP（Model Context Protocol）是 Anthropic 于 2024 年底开源的协议，目标只有一件事：给「模型应用 ↔ 工具/数据源」这条链路定一个统一接口。可以类比为 AI 应用层的 USB-C。

## 它解决的核心问题：M×N → M+N

没有统一协议时，M 个客户端（Agent、IDE、桌面助手）要接 N 个工具（数据库、工单系统、代码仓库……），最坏情况要写 M×N 份适配代码，每份都重复处理鉴权、参数校验、错误格式。MCP 把两端各自收敛：客户端实现一次 MCP client，工具方实现一次 MCP server，其余组合即插即用。

协议定义了三类能力：

- **Tools**：模型可主动调用的函数（查询、执行动作）
- **Resources**：应用侧可控的上下文数据（文件、配置）
- **Prompts**：用户侧触发的模板

传输层常用两种：**stdio**（本地子进程，最简单）和 **Streamable HTTP**（远程部署）。

## 最小可用路径

用 Python SDK 写一个本地工具 server，十几行就能跑通：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("ops-tools")

@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查询订单状态，返回状态与最近一次物流事件。"""
    return db.get_order_status(order_id)

mcp.run()  # 默认 stdio 传输
```

接入步骤：

1. **实现 server**：一个函数一个 `@mcp.tool()`，docstring 就是给模型看的说明书
2. **注册到 OpenClaw 的 MCP 配置**：stdio 场景就是一条启动命令，远程场景填 URL
3. **验证**：用 MCP Inspector 或直接让 Agent 调一次，确认参数传递与返回格式
4. **观察 token 消耗**，再决定是否接入第二个工具

## 踩坑点

- **工具描述就是 prompt**。模型靠 docstring 和参数描述决定调不调、怎么调。写「查询数据」这种模糊描述，错调率会明显上升。
- **工具不是越多越好**。每个已连接 server 的 schema 都会注入上下文，挂十几个 server 后 system prompt 直接膨胀，成本和准确率双双恶化。
- **stdio 环境问题**。server 以子进程运行，npx/uvx 找不到、Python venv 不对、PATH 不一致，是本地接入最高频的故障来源。
- **返回值要小**。工具把整张表丢回给模型，上下文立刻爆炸。截断、分页、摘要都应在 server 侧完成。
- **安全别裸奔**。MCP server 以你的权限运行，工具返回内容也可能夹带注入指令。三方 server 先审计，写操作默认走人工确认。

## 可复用建议

- 一个 server 只管一个领域；工具粒度「够用就粗」，5 个描述清晰的工具好过 20 个细碎的
- 写操作拆成「预览 + 确认」两步，配合审批流
- server 尽量无状态，返回紧凑 JSON，超长输出主动截断
- 锁定 server 版本；协议仍在迭代（如 SSE 传输已演进为 Streamable HTTP），升级前跑回归
- 记录每次工具调用的入参、出参、耗时，排障时省一半时间

## 总结

MCP 并没有发明新能力，它把「Agent 接工具」从每对组合手写胶水，变成两端各实现一次标准接口。对 OpenClaw 用户而言，实际收益是工具可复用、可审计、可独立迭代。建议从一个 stdio server、两三个工具起步，先把描述和返回值打磨好，再考虑远程部署。协议本身不复杂，复杂的是工具设计的品味——那才是真正拉开差距的地方。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/b260431b9d7f5e18.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/52d859235824d78f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/e210cf9a4112b048.png)

