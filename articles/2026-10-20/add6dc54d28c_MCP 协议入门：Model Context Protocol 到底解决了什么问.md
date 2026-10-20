---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42127
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标是给 LLM 应用和外部工具、数据源之间定一个统一的连接标准。社区常说它是"AI 应用的 USB-C 接口"——这个比喻不算夸张，但要理解它到底省了什么事，得先看它出现之前的局面。

## 问题：M×N 的集成灾难

假设有 M 个 AI 客户端（IDE、聊天助手、Agent 框架），N 个想被调用的服务（GitHub、数据库、内部 API）。没有统一协议时，每对组合都要写一份定制胶水代码：客户端各自定义 function calling 格式，服务方为每个客户端写适配，维护成本是 M×N，且没有一处能生态复用。

MCP 把它压缩成 M+N：工具方实现一次 MCP Server，客户端实现一次 MCP Client，中间靠协议对齐。你的 Agent 框架只写一次接入逻辑，就能吃掉所有现成 Server；反过来，你给内部系统写一个 Server，所有支持 MCP 的宿主都能直接用。

## 做法：最小可用路径

1. 理清三个核心原语：**Tools**（模型主动调用的动作）、**Resources**（可读取的数据）、**Prompts**（预置模板）。日常自动化九成场景只用 Tools。
2. 用官方 SDK（Python/TypeScript）起一个最小 Server，核心就是注册带 schema 的工具函数：

```python
@mcp.tool()
def query_order(order_id: str) -> str:
    """按订单号查询订单状态，返回含 status/amount/eta 字段的 JSON。"""
    ...
```

3. 传输方式二选一：本地场景用 stdio（宿主以子进程拉起），远程服务用 Streamable HTTP。
4. 用 MCP Inspector 或直接接入宿主做端到端验证，确认工具列表、调用、返回三段链路都通。

## 踩坑点

- **docstring 不是注释，是提示词。** 工具描述含糊，模型要么不调用，要么乱传参。这是新手最容易低估的一环。
- **返回值要克制。** 把整张表 dump 进上下文，一次调用烧掉几千 token。分页、截断，只返回模型做决策需要的字段。
- **工具热更新基本是伪需求。** Server 运行时增删工具，多数宿主不会自动刷新工具列表，重启最稳。
- **信任边界别糊。** 第三方 Server 的工具描述本身就是注入面，接入前过一遍代码；工具返回内容里若混入指令文本，宿主要当数据处理，不当命令执行。
- **破坏性操作不要裸暴露。** 删除、支付、发消息这类工具，至少加确认层或参数二次校验。

## 可复用建议

- 工具设计原则：**少而粗，语义清晰**。十个意图模糊的细粒度工具，不如三个意图明确的。
- 错误信息写给模型看：`order_id 不存在，请确认是否含多余空格` 远比 `Error 500` 有用。
- 先做只读 Server 跑通全链路，再逐步放开写操作。
- 给每次工具调用记日志（入参、返回摘要、耗时），排查 Agent 异常行为时，这是唯一可靠的证据链。

## 总结

MCP 解决的不是模型能力问题，而是集成工程问题：把 M×N 压成 M+N，让工具生态有了统一插槽。但协议本身很薄，真正的工程量在工具面的设计——描述怎么写、返回怎么裁、权限怎么收。把这层做扎实，你的 Agent 才能从"能调工具"进化到"调得对工具"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/71dd44edac7ac65b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/f57a0b6199e23377.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/a1bbb669ee55c42f.png)

