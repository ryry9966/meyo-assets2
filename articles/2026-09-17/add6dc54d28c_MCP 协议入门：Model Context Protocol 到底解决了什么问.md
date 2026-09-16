---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 37910
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

过去一年做 Agent 落地，最耗时的往往不是模型部分，而是“接线”：让模型能查数据库、调内部 API、读本地文件。每接一个数据源，就要写一遍函数定义、参数校验、鉴权和错误处理；换个模型或框架，这套胶水代码还得重写。MCP（Model Context Protocol）就是针对这个痛点的开放协议，2024 年底由 Anthropic 开源，目前主流 IDE 和 Agent 框架都已提供客户端实现。

## 问题：M×N 的集成困境

没有统一标准时，M 个应用对接 N 个工具，复杂度是 M×N——每个应用自定义工具格式，互不兼容；工具提供方要为每个宿主单独维护接入代码。MCP 把两端解耦成 M+N：应用实现一次 MCP Client，工具实现一次 MCP Server，中间用统一的 JSON-RPC 2.0 消息通信。传输层本地走 stdio，远程走 Streamable HTTP。

## 做法：三步跑通最小闭环

1. **理清三个原语**：Tools（模型决定何时调用，用于“做事”）、Resources（应用决定注入什么上下文，用于“读数据”）、Prompts（用户触发的模板）。多数自动化场景只需要 Tools。
2. **用官方 SDK 写最小 Server**。Python 侧 `pip install "mcp[cli]"`，用 FastMCP 装饰器暴露一个函数即可；函数 docstring 和类型标注会自动转成模型看到的 schema。
3. **在宿主中注册**。Claude Desktop 或自建 Agent 的配置里加一行 stdio 启动命令即可。调试用官方 MCP Inspector，能看到完整握手和工具调用报文。

## 踩坑点

- **stdio 模式下，Server 里所有 `print` 都会污染 stdout 的 JSON-RPC 流**，导致宿主解析失败。日志一律写 stderr。
- **工具描述写得含糊，模型就会选错工具或编造参数**。描述是给模型看的接口文档，要写清适用场景、参数约束和返回结构。
- **一次挂载几十个工具**，context 膨胀且选择错误率明显上升。按会话动态启用子集，或做一层聚合入口。
- **工具返回值直接进入上下文，是 prompt injection 的高危入口**。对外部数据源的返回内容做截断和过滤，敏感操作加人工确认环节。
- **远程部署注意协议版本**：早期 HTTP+SSE 传输已被 Streamable HTTP 取代，新旧客户端混用时报错多半出在这里。

## 可复用建议

- Server 保持小而单一，组合逻辑放在宿主层，不要写大而全的“万能 Server”。
- 返回结构化、精简的结果；大文件、长列表用 Resource 引用，别直接塞进工具返回值。
- Server 尽量无状态，便于横向扩展；确需会话状态时用 Streamable HTTP 的 session 机制。
- 把工具描述当代码维护：改接口必改描述，每次发版用 Inspector 做回归验证。

## 总结

MCP 本质上不是智能技术，而是一份接口约定：它把“模型如何发现工具、调用工具、拿回结果”标准化，让应用与工具两侧可以独立演进，把胶水代码收敛为一次实现、多端复用。它解决的是工程问题，不是模型能力问题——工具最终好不好用，仍取决于接口设计和返回数据的质量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/201414ffe6a9a078.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/dee17f4bf8dbd8b7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/ccabb5eba1fd23d7.png)

