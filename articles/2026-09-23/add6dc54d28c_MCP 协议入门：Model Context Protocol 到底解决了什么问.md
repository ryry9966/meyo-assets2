---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38550
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

做 Agent 的人都绕不开一件事：让模型调用外部工具。查数据库、读文件、调内部 API、发消息……一两个工具时手写 function call 还能忍，但工具一多、客户端一多，集成成本就开始爆炸。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开放的一套协议，目标是把「模型 ↔ 工具/数据源」这一层标准化。可以类比 USB-C：工具方（MCP Server）按统一接口插上来，宿主（MCP Client/Agent）按统一协议消费。

## 它到底解决什么问题

核心是经典的 **M×N 集成问题**：

- 没有 MCP：M 个 Agent 客户端要接 N 个工具，最坏要写 M×N 份胶水代码，每家的鉴权、传输、参数格式、错误处理都不一样。
- 有了 MCP：工具方写 1 个 MCP Server，客户端实现 1 套 MCP Client，问题降级为 **M+N**。

此外它顺带统一了几件事：

1. **工具发现**：客户端启动时 list tools，模型看到的是带 JSON Schema 的标准描述，不用每家自定义 prompt 片段。
2. **三类原语边界清晰**：tools（模型主动调用、可有副作用）、resources（只读上下文、可注入对话）、prompts（预置模板化交互）。新手常混用，其实分工很明确。
3. **传输层标准化**：本地 stdio、远程 Streamable HTTP，消息格式统一走 JSON-RPC 2.0。

## 在 Agent 项目里怎么用（最小步骤）

1. **确认宿主能力**。看你的 Agent 框架（OpenClaw 插件体系、自家 runtime）是否有 MCP client 支持；没有就先接官方 SDK（TS/Python 都有）。
2. **从现成 Server 起步**。filesystem、SQLite、fetch 这类官方 Server 足够验证链路，先跑通再自研。
3. **本地调试用 stdio**：配置里写清 command/args/env，客户端负责拉起子进程；远程服务再切 Streamable HTTP。
4. **自研 Server 时只暴露真正需要的 tools**，每个 tool 写清 description 和输入输出 Schema——description 是给模型看的，不是给人看的。
5. **记录每次 tool call 的入参和结果**。排查「模型为什么调错工具」时，这是唯一可靠的证据。

## 踩坑点

- **description 含糊**，模型就会选错工具或编造参数。宁可啰嗦，别省字。
- **工具列表太长**会吃 context，也拉低选择准确率。单个 Server 控制在十几个 tools 以内，能合并就合并。
- **stdio 模式的子进程管理**：注意工作目录和环境变量；Server 崩溃后要有重启策略，别让整个 Agent 会话跟着挂。
- **安全别大意**：MCP Server 拿的是 Agent 的权限。第三方 Server 返回的 resources 内容可能携带注入指令，进入上下文前要过一遍你的过滤策略。
- **Schema 要严格**：nullable、required、枚举值对齐 JSON Schema 规范，模型对宽松 Schema 的容错比你想象的差。

## 可复用建议

- 把 MCP Server 当独立服务管理：独立仓库、独立版本、独立鉴权，别和业务代码耦死。
- 有副作用的 tool（写库、发消息）做幂等设计或二次确认机制。
- 团队内沉淀一份「内部工具 MCP 化」模板：脚手架 + 日志 + 错误码约定，新人半天能上线一个新 Server。

## 总结

MCP 没有魔法，它只是把「Agent 接工具」从各家一套私货变成一个公开协议，价值在于生态复用：工具写一次，所有支持 MCP 的客户端都能用。建议路径是——先用现成 Server 验证链路，再逐步把内部工具 MCP 化，最后补齐安全与可观测性。协议本身很薄，真正的工程量在描述质量、权限边界和稳定性上。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/7f190c9e523da515.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/7e8498c9335765a3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/a94a463c17eb09b7.png)

