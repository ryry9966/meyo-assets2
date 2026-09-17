---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38033
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

接大模型做 Agent，绕不开一个问题：模型本身只会生成文本，真正干活要靠外部工具。读写数据库、调内部 API、操作文件系统……这些"手"怎么接上去，过去一直没有统一答案。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标是把"模型如何调用外部能力"这件事标准化。目前主流 Agent 框架和 IDE 都已支持，在 OpenClaw 的插件与自动化体系里，它也是最常用的接入方式之一。

## 它解决的真实问题

没有 MCP 时，这是典型的 M×N 问题：M 个宿主（Host）要接 N 个工具/数据源，每个组合都得写定制对接代码。你写了个抓取工具，只能在自家脚本里用，换一个宿主就得重写。

MCP 的做法是把中间层拆出来：

- **Host**：跑模型的客户端（Agent 框架、IDE 等），内置 MCP Client
- **Server**：独立的轻量服务，声明自己提供哪些 tools / resources / prompts
- **传输层**：本地用 stdio，远程用 Streamable HTTP

Server 只需实现一次协议，任何支持 MCP 的 Host 都能直接挂载。核心价值一句话：**把集成成本从 M×N 降到 M+N**。

## 实际做一个 MCP Server 的步骤

以 Python SDK 为例：

1. 安装 SDK，用 FastMCP 定义 server；
2. 用 `@mcp.tool()` 暴露函数，写清楚参数 schema 和描述；
3. 本地先跑 stdio 传输，用官方 Inspector 工具手测每个工具的调用与返回；
4. 挂到目标 Host 做端到端验证；
5. 稳定后再迁到 Streamable HTTP + 鉴权，供团队复用。

## 踩坑点

- **stdout 污染**：stdio 模式下 stdout 只能走 JSON-RPC，`print` 调试日志会直接把会话打挂，日志一律走 stderr。
- **工具描述敷衍**：描述是模型选工具的唯一依据。"do something" 式的描述会让模型乱调或拒绝调用，请当成写给陌生人的 API 文档来写。
- **工具太多**：一次注入几十个工具，上下文膨胀、误选率上升。合并同类工具，用一个带 `action` 参数的工具替代一组细粒度工具，往往效果更好。
- **返回值过大**：工具把整张表吐回去，几轮就把上下文吃满。务必做分页、截断、摘要。
- **协议版本不匹配**：Host 与 Server 版本差异会导致能力协商失败，升级 Host 后记得回归测试旧 Server。
- **安全**：不要随意加载来源不明的 Server；工具返回内容本身也是注入入口，写操作要有确认机制和幂等设计。

## 可复用的建议

- 先在本地 stdio + Inspector 跑通，再考虑远程化，不要一上来就折腾鉴权和部署；
- 每个 Server 聚焦一个领域，宁可多个小 Server，不要一个巨型 Server；
- 给所有工具调用记结构化日志，排查"模型为什么这么调"全靠它；
- 工具命名带领域前缀（如 `github_create_issue`），降低跨 Server 撞名与误选概率；
- 写操作工具返回操作 ID 和结果状态，方便宿主层做确认与回滚。

## 总结

MCP 本身并不神秘，它解决的是集成标准化这个老问题，类似 USB-C 之于外设。协议层很简单，难点在工具设计：描述怎么写、返回怎么裁剪、权限怎么收。协议是固定的，**工具质量才是 Agent 能力的上限**。建议从把一个内部 API 包成 MCP Server 开始，跑通全链路后再谈规模化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/821c67e3967f833b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/ccb918f641d8b35e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/f3f8480d96598def.png)

