---
title: MCP 协议入门：从 M×N 到 M+N，Model Context Protocol 到底解决了什么
feedId: 39012
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景：接入比模型更麻烦

做过 Agent 或自动化的人大概都有体会：模型本身不难接，难接的是"周边"。让它读表格、查内部数据库、操作浏览器、调内部 API——每接一个工具，就要为当前这套 Agent 框架手写一份 glue code，换个框架基本作废。

这就是典型的 M×N 问题：M 个应用 × N 个工具/数据源，最坏要写 M×N 份集成。Function calling 缓解了一部分，但它只解决"模型会说我要调什么"，工具怎么被发现、参数怎么协商、连接怎么建立，各家仍是各玩各的。

MCP（Model Context Protocol，Anthropic 2024 年底开源的协议）做的事，本质上是把 M×N 拆成 M+N：工具方实现一次 MCP Server，任何支持 MCP 的 Host 都能直接用。

## 它具体标准化了什么

MCP 基于 JSON-RPC 2.0，核心只有三类原语：

- **Tools**：模型可主动调用的动作（执行函数）
- **Resources**：可读取的上下文数据（文件、记录，偏只读）
- **Prompts**：预置的提示词模板

加上握手时的能力协商与工具发现（先 initialize，再 list tools，再 call），主流 transport 两种：本地 stdio，远程 Streamable HTTP。协议面很小，这正是它容易被采纳的原因。

## 最小实践步骤

以给 Agent 加"查内部工单"能力为例：

1. 用官方 SDK（Python/TypeScript）起一个最小 Server，暴露一个 tool：`search_tickets(query, limit)`
2. **重点写好 tool 的 description**——这是给模型看的，不是给人看的 API 文档。讲清楚"什么场景该用我、参数格式、返回结构"
3. 先用 MCP Inspector 本地调通 schema 和返回
4. 在 Host 侧 MCP 配置里注册该 server（stdio 模式一条 command 即可）
5. 跑通后再搬远程：换 Streamable HTTP，加鉴权

## 踩坑点

- **stdout 污染**：stdio 模式下 stdout 是协议通道，任何 print/调试日志打进去都会让 JSON-RPC 帧错乱。日志一律走 stderr。
- **描述写太烂**：模型选错工具、传错参数，九成是 description 的问题，不是模型的问题。
- **返回体过大**：一次吐几千行 JSON 会直接吃掉上下文窗口。默认分页、做摘要、强制 limit。
- **长任务阻塞**：超过 Host 超时的工具，要么拆成多步，要么上报 progress notification。
- **信任边界**：MCP Server 拿着你的凭据在跑，第三方 server 等于供应链风险；工具描述本身也可能被注入（tool poisoning）。生产环境别接来路不明的 server。

## 可复用建议

- 按领域拆 server，单个 server 只放少量高内聚 tool——工具列表太长会稀释模型注意力。
- 只读数据用 Resources，动作用 Tools，别把查询包装成"调用"。
- 工具设计成幂等的；错误信息要能帮模型自我修正，比如"date 格式应为 YYYY-MM-DD"，而不是笼统的"参数错误"。
- 顺序别反：先 Inspector 测试，再接 Host，最后才上远程 transport。

## 总结

MCP 不会让模型变强，它解决的是"接入"这件事：把一次性的 glue code 变成标准契约。单看一个 server 收益有限；当你有多个 Agent、多个工具、多个框架需要互相组合时，M+N 对 M×N 的优势才真正显现。对 OpenClaw 这类插件化生态来说，它恰好是插件间互操作的公共底座——值得在写下一个自定义集成之前，先花五分钟想一想。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/0d51cc6d16788ae3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/276b256f6f55468f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/a99bbb24fd3d4663.png)

