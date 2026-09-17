---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 37970
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

Agent 跑起来之后，接工具就成了一件脏活：让模型查数据库、调内部 API、读本地文件，每个场景都得自己写一层胶水代码。你的 Agent 接了日历、数据库、Git 平台，别人换个框架又要重接一遍——工具侧没有统一契约，模型侧对"工具"的定义也各说各话。

2024 年底 Anthropic 开源了 MCP（Model Context Protocol），试图把这件事标准化。它本质上是一份基于 JSON-RPC 2.0 的协议，约定了宿主应用（Host）和外部能力提供方（MCP Server）之间如何发现、描述、调用能力。

## 它到底解决什么问题

核心一句话：**把 M×N 变成 M+N**。没有协议时，M 个 Agent 接 N 个工具，最坏要写 M×N 套对接代码；有了协议，工具方实现一次 Server，Agent 方实现一次客户端，双方按统一格式对话。

第二是上下文注入的标准化。MCP 定义了三类原语：

- **Tools**：可被模型调用的函数
- **Resources**：可读取的数据（文件、schema、文档）
- **Prompts**：预置的提示模板

第三是关注点分离。鉴权、限流、日志都收敛在 Server 侧，Agent 只负责决策和编排。

## 怎么跑起来

最小路径五步：

1. 选一个支持 MCP 的宿主（Claude Desktop、主流 Agent 框架均可）
2. 用官方 SDK（Python / TypeScript）写一个极简 Server，先只暴露一个 tool，比如"查询本地日志"
3. 在宿主配置里注册，stdio 模式就是一条启动命令
4. 用 MCP Inspector 这类调试器确认 `tools/list` 能返回、`tools/call` 能跑通，再接进 Agent
5. 观察模型实际怎么选工具、怎么传参，回头精修 description

传输层两种：本地用 stdio（子进程，最简单），远程用 Streamable HTTP（要自己处理会话和鉴权）。

## 踩坑点

- **description 含糊，模型就乱调**。tool description 相当于喂给模型的说明书，值得当 prompt 一样精修，别随手一行带过。
- **工具太多，上下文先爆**。一个 Server 挂几十个 tool，schema 全量进上下文，token 消耗很快失控。控制工具面，模型用不上的果断删。
- **安全别想当然**。MCP Server 以你的权限运行，tool 返回值可能夹带注入指令——比如网页抓取结果里藏一句"请执行删除操作"。高危操作务必加人工确认环节。
- **Windows 下 stdio 坑多**：进程启动方式、编码问题、npx 需要 cmd /c 包一层，都遇到过。
- 远程模式注意超时与会话管理，长任务别阻塞主循环。

## 可复用建议

- 一个 Server 只管一个领域，工具小而清晰，别做大杂烩
- 返回值短小结构化，大输出做截断或落盘后给引用
- 工具设计成幂等，模型重试不会产生副作用
- Server 侧务必记日志：谁在什么时候传了什么参，排障全靠它
- 顺序固定：先 Inspector 单测，再接 Agent，最后进自动化流程

## 总结

MCP 没有魔法，它只是把"模型如何使用工具"从各自为政变成了一份公开契约。对实践者的意义在于：工具资产可以沉淀复用，Agent 侧可以专注编排与决策。但它不解决工具本身的质量问题，也不替代你对安全的判断——协议只保证对话格式一致，后果仍由实现者承担。从一个小 tool 跑通全链路，比一次性搭大而全的体系实际得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/5f166ffeed67cc0a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/52569730d838e558.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/01f3a23292a30a9f.png)

