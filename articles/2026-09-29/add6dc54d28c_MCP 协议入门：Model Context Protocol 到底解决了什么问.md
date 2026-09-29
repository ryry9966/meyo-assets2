---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 39652
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

过去一年做 Agent 的同学很难绕开 MCP（Model Context Protocol）。它是 Anthropic 在 2024 年底开源的协议，目标很朴素：给"模型/Agent 如何连接外部工具和数据"定一个统一标准。对 OpenClaw 这类支持插件扩展的 Agent 框架，以及各种自动化工作流场景，MCP 已经是事实上的接入层选择。

## 它到底解决了什么问题

MCP 之前，接工具是纯手工作坊：你有 M 个 Agent 应用、N 个能力（数据库查询、工单系统、本地文件、搜索 API……），每个应用接每个能力都要单独写适配——Tool schema 格式不同、传输方式不同、鉴权方式也不同，最坏情况是 M×N 份胶水代码，且彼此语义对不上。

MCP 把问题压缩成 M+N：工具方按协议实现一次 Server，应用方实现一次 Client，中间用 JSON-RPC 通信。几个核心角色：

- **Host/Client**：宿主应用（Agent 运行时），内嵌 Client 与 Server 通信；
- **Server**：能力提供方，暴露三类原语——Tools（可调用动作）、Resources（可读取数据）、Prompts（可复用模板）；
- **Transport**：本地用 stdio，远程用 Streamable HTTP（早期版本是 HTTP+SSE，注意差异）。

## 上手步骤

1. 用官方 SDK（Python 的 FastMCP 或 TS SDK）写一个最小 Server，先只暴露一个只读工具，比如"按 ID 查订单"；
2. 用 MCP Inspector 单测，确认 schema 和返回结构没问题，再接进 Agent；
3. 在 OpenClaw 的工具配置里注册该 Server（stdio 写启动命令，远程写 URL）；
4. 观察模型选工具的行为，迭代工具名和 description；
5. 稳定后再加写操作，并补上确认与审计。

## 踩坑点

1. **description 不是注释，是提示词。** 写得像接口文档，模型就会选错工具或拒绝调用。要写清楚何时该用/不该用、输入示例、失败时返回什么。
2. **工具数量失控。** 一个 Server 塞几十个工具，上下文爆炸、选择混乱。按业务域拆分，单个 Server 控制在个位数到十来个。
3. **返回值过大。** 把整张表吐给模型是灾难。过滤、分页、摘要都应在 Server 端做，只回必要字段。
4. **权限与安全。** stdio Server 与宿主同权限运行，别让它直接包一层有高危权限的脚本；写操作要加确认或设计成可回滚。
5. **协议版本演进。** SSE 到 Streamable HTTP、鉴权机制都在变，SDK 版本要钉死，升级前先读 changelog。

## 可复用建议

- 工具设计对齐用户意图，而非照搬内部 API：`query_order_by_id` 优于裸露的 REST 路径；
- 先只读、后写入，破坏性操作加二次确认；
- Server 端记录每次调用的入参出参，出问题能回放；
- 在 description 里写返回结构示例，模型填参准确率会明显提升。

## 总结

MCP 解决的是"连接标准化"，不是"工具质量"。它帮你收敛胶水代码，但工具是否好用，仍取决于 description 质量、返回数据的裁剪和安全边界的设计。把 MCP 当协议层的事实标准来用，把工程功夫花在工具设计本身，才是正确姿势。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/021ec583cf51edf1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/e3eb15de5953ef70.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/dff09bc3fb10cc50.png)

