---
title: MCP 协议入门：把 M×N 的集成问题变成 M+N
feedId: 41160
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景：工具接入的“手工时代”

大模型本身只会生成文本，Agent 真正干活靠的是外部工具和数据。在 MCP 出现之前，每个应用对接每个工具都是手写胶水代码：定义 function calling 的 schema、处理鉴权、解析返回、兜底报错。值得注意的是，function calling 只标准化了“模型↔应用”这一段，“应用↔工具”那一段依然各自为政——你在 A 应用里接了 GitHub，换到 B 应用还得再写一遍。

## 问题：M×N 的集成成本

这在数学上很直白：M 个应用 × N 个数据源 = M×N 份集成代码。工具方没动力为每个宿主写适配，应用方也无法穷举所有工具，生态因此碎片化。

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），目标就是把 M×N 压成 M+N：工具方写一次 MCP Server，任何实现了 MCP Client 的应用都能直接用。

## 做法：一套协议，三个原语

MCP 本质是基于 JSON-RPC 2.0 的客户端-服务器协议，架构分三层：

- **Host**：Agent/应用本体（Claude Desktop、IDE、OpenClaw 这类自托管 Agent 都算）
- **Client**：Host 内部与单个 Server 保持 1:1 连接的中间层
- **Server**：暴露能力的独立进程

Server 对外暴露三类原语：**Tools**（模型可主动调用的动作）、**Resources**（可读取的数据）、**Prompts**（预置提示词模板）。传输层最常用两种：本地进程走 stdio，远程服务走 Streamable HTTP。

写一个最小 Server 大概十行（Python SDK）：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("order-tools")

@mcp.tool()
def get_order(order_id: str) -> dict:
    """按订单号查询订单状态，返回状态与金额"""
    return db.fetch_order(order_id)

mcp.run()  # 默认 stdio 传输
```

在 Host 配置里声明启动命令和 env，Agent 启动时会自动发现工具列表并注入给模型——调用侧代码一行不用改。

## 踩坑点

1. **工具描述是写给模型看的 API 文档。** 描述含糊，模型就会传错参数、在不该调用时调用。花在 docstring 上的时间比花在实现上更值。
2. **stdio 模式下别往 stdout 打印日志。** stdout 是 JSON-RPC 通道，一行 print 就能让会话挂掉，日志一律走 stderr。
3. **返回数据别贪多。** 把整张表塞进 tool result 等于亲手撑爆上下文窗口。做分页、限制字段，必要时返回引用让 Agent 按需再取。
4. **写操作必须加确认层。** 模型会犯错，也会被工具返回内容里的注入指令误导——prompt injection 可以来自工具结果。删除、支付类工具默认要求人工确认。
5. **长任务别让 Agent 干等。** 返回 task_id 让它轮询，否则超时重试会把对话搞乱。
6. **注意协议版本。** MCP 规范迭代很快（2024-11-05 → 2025-03-26 → 2025-06-18），Client 和 Server 的 SDK 版本差太远会导致能力协商失败。

## 可复用建议

- **从只读工具起步**，先跑通 discovery → 调用 → 回传的闭环，再逐步放开写操作。
- **一个 Server 收敛一个领域**（db、jira、内部 API 各一个），小而可组合，别造“万能 Server”。
- **工具粒度对齐意图而不是 REST 端点。** 10 个语义清晰的工具好过 50 个薄封装——工具越多，模型选错的概率越高。
- **把每次调用的入参、出参、耗时落日志**，排查 Agent 行为问题时这是唯一可靠的证据。
- **错误信息写给模型看**：报错里直接说明缺什么参数、该怎么改，模型能自我修正，人就不用介入。

## 总结

MCP 没有让模型变聪明，它只是把集成成本从 M×N 降到 M+N，让“接一个新工具”从开发任务变成配置任务。协议本身很薄，真正的工作量在工具粒度设计、描述质量和安全边界——这三件事没有标准答案，值得按自己的场景慢慢打磨。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/5f5476576c0599db.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/9cc892dd4f2457ff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/46cf967e9d461da1.png)

