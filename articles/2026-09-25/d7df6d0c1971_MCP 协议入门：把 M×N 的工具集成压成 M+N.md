---
title: MCP 协议入门：把 M×N 的工具集成压成 M+N
feedId: 38844
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），一年多下来，它基本成了 Agent 生态里"工具接入"的事实标准。但社区里对它的理解两极分化：要么当成万能银弹，要么觉得只是又一个 JSON 格式。这篇从工程视角聊聊它到底解决了什么、没解决什么。

## 问题

在 MCP 之前，给 Agent 接工具是个典型的 M×N 问题：M 个 Agent 框架（各自的 function calling 格式、插件体系），N 个工具和数据源（数据库、文件系统、内部 API）。想打通任意组合，就得写 M×N 个适配器。更麻烦的是上下文：文件怎么喂给模型、检索结果以什么格式注入、超长输出怎么截断，每家都自己定义一遍。

MCP 把它压成 M+N：Agent 宿主程序实现一次 MCP 客户端，工具侧实现一次 MCP 服务器，中间用 JSON-RPC 通信。协议定义了三类原语：Tools（供模型调用的动作）、Resources（供应用注入的数据）、Prompts（预置提示模板）。传输层主流两种：本地进程走 stdio，远程服务走 Streamable HTTP。

## 做法

以"让 Agent 查询内部 PostgreSQL"为例：

1. **先找现成的**：数据库、文件系统、浏览器这类常见场景，社区基本都有维护良好的 MCP server，优先复用；实在没有再用官方 SDK（Python / TypeScript / Go 等）写。
2. **单独跑通**：先用 MCP Inspector 测 server，确认 `tools/list` 和 `tools/call` 都正常，再接进 Agent。这一步能省掉大半排障时间。
3. **配置接入**：本地 server 配 stdio 命令和环境变量；远程 server 配 URL 和鉴权头。
4. **控制工具面**：不要把 server 的所有 tools 全挂上，挑用得到的，并逐个核对 description 和参数 schema。
5. **加观测**：上线前在 server 侧记录每次调用的工具名、参数、耗时和返回摘要，出问题才有的查。

## 踩坑点

- **stdio 模式下 stdout 只能走协议消息**。调试日志打到 stdout 会污染 JSON-RPC 流，客户端表现为"一连就断"。日志一律走 stderr。
- **tool description 是模型上下文的一部分**。写得含糊，模型就选错工具或传错参数；工具挂太多，token 消耗和误触发率同步上涨。
- **协议版本有漂移**。早期 HTTP+SSE 传输已被 Streamable HTTP 取代，客户端和服务端版本要对齐，锁版本比追新稳。
- **长耗时工具要处理超时和进度上报**，否则客户端默认超时会直接掐断调用。
- **安全别裸奔**：server 拿到的凭证和 Agent 权限相当，工具描述和 Resource 内容可能携带注入内容。高危工具走人工确认，凭证按 server 隔离。

## 可复用建议

- 按"一个域一个 server"拆分：文件、数据库、搜索各自独立，粒度和权限都好控。
- 工具设计参照 API 设计：足够粗让模型一步到位，足够细让危险操作可单独审批。
- 返回值做裁剪：大结果默认截断，翻页和过滤做成工具参数，别让十万字查询结果直接灌进上下文。
- description 按"写给模型看的 prompt"来写，而不是写给人看的文档。

## 总结

MCP 没有让工具变可靠，也没有替你做安全，它解决的是"工具怎么接、上下文怎么传"这一个标准化问题，但这恰恰是 Agent 工程里最脏、最重复的部分。把它理解成工具侧的 USB-C 比较准确：接口统一了，设备本身好不好用，仍然取决于你的工程质量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/b20c247ac8bf740d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/21dc4933c217df21.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/46e07cd27f39d55b.png)

