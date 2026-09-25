---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38945
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），一年多过去，它基本成了 Agent 接入外部工具与数据的事实标准。OpenClaw 这类可自托管的 Agent 框架，对 MCP 的支持也已经比较完整。但很多人只是照抄配置，未必想清楚它到底替代了什么。

## 问题：N×M 集成困境

没有 MCP 之前，让 Agent 用上外部能力通常两条路：要么用 function calling 自己定义 schema、自己写鉴权和胶水代码；要么接入各平台的私有插件体系。结果就是 M 个工具 × N 个应用 = M×N 份集成代码，给这个 Agent 写的接入，换一个壳就得重写。

MCP 把这层拆开：工具方实现一次 MCP Server，应用方实现一次 MCP Client，中间用统一的 JSON-RPC 消息通信。N+M 次实现替代 N×M 次。

协议里最常用的是三个原语：

- **Tools**：模型可调用的动作，通常有副作用；
- **Resources**：可读取的上下文，如文件、数据行；
- **Prompts**：预置的提示模板。

传输层两种：本地 stdio（起子进程）和远程 Streamable HTTP（旧的 SSE 接法已废弃）。

## 做法：最小接入步骤

1. **选现成 Server**。官方和社区维护了 filesystem、GitHub、SQLite、Playwright 等，先用它们验证整条链路。
2. **配置注册**。在 OpenClaw 的 MCP 配置里填 `command + args`（stdio）或 `url`（HTTP），重启会话。
3. **验证工具列表**。列出该 server 暴露的 tools，核对参数 schema 是否符合预期。
4. **小步调用**。让 Agent 调一个只读工具，确认参数解析和返回结构没问题。
5. **有必要再自己写**。用官方 SDK（Python/TypeScript）包装内部 API，几十行就能起一个 server。

## 踩坑点

- **工具描述决定调用质量**。模型完全靠 name 和 description 选工具，写得含糊就会被误调或漏调。description 是给 LLM 读的，不是给人读的。
- **工具太多撑爆上下文**。一次挂十来个 server、上百个工具，定义本身就吃掉大量 token，还会降低选择准确率。按需开关。
- **返回数据太肥**。整张表不如带条件的查询。分页和裁剪要在 server 侧做，别指望模型自己过滤。
- **stdio 的工作目录**。子进程式 server 对 cwd 敏感，路径全配绝对路径，能少踩一半坑。
- **安全别裸奔**。第三方 server 等于给模型开新权限，先只读、再限白名单，警惕工具描述里藏 prompt injection（tool poisoning）。
- **传输协议混用**。照老教程用 SSE 连新 server 会直接 404，认准 Streamable HTTP。

## 可复用建议

- **先只读后写**：新 server 接入后，只用查询类工具跑一阵，再放开写操作。
- **薄封装优于通用化**：给内部系统写 server 时，暴露 5 个贴合业务的粗粒度工具，好过 50 个标准 CRUD。
- **配置进版本库**：MCP server 列表是团队环境的一部分，跟着仓库走，别散落在各人本地。
- **全程打日志**：MCP 消息是明文 JSON-RPC，把请求响应记下来，排障效率高一个量级。

## 总结

MCP 并不神秘：它没有让模型变聪明，只是把“应用如何拿到工具”这件事标准化了，真正的价值在生态——工具写一次到处用，Agent 换壳不换集成。判断是否该用很简单：接入点超过两个、且会跨应用复用，就值得收敛到 MCP；如果只是一个 Agent 调一个内部接口，直接 function calling 反而更省事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/e0f4b0845408a446.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/0523fc6fba29301c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/3aa0fd8e7b51a20d.png)

