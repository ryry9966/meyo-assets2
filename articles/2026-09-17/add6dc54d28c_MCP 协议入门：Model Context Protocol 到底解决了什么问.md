---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 37982
source: 综合讨论
publishedAt: 2026-09-17
---

## 背景

做 Agent 自动化的团队几乎都撞过同一堵墙：模型本身能力不差，但它是“隔离”的——不知道你数据库里有什么、调不了你内部的脚本、读不了本地文件。要让 Agent 真正干活，就得给它接外部能力，而“接”这件事长期没有统一标准。

结果是每个框架造一套自己的插件格式，每个工具写一遍适配：A 框架一份 tool schema，B 客户端一套调用约定。3 个 Agent 接 8 个工具，就是 24 份胶水代码。这就是典型的 M×N 集成问题。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的标准协议，思路很直接：**把 M×N 压成 M+N**。工具方只需实现一次 MCP Server，所有支持 MCP 的宿主（Agent、IDE、桌面客户端）都能直接用；宿主方只需实现一次 MCP Client，就能接入整个生态。协议底层是 JSON-RPC 2.0，传输层常见 stdio（本地子进程）和 Streamable HTTP（远程服务）两种。

## 它到底解决了什么

拆开看是三件事：

1. **统一的工具发现与调用**：Server 声明有哪些 tools（含 JSON Schema 参数定义），Client 拉取列表后注入模型上下文，模型按需调用。
2. **资源暴露（Resources）**：文件、数据库 schema、配置这类只读上下文，不必包装成“工具”，以资源形式被宿主读取即可。
3. **提示模板（Prompts）**：Server 可顺带发布常用提示词模板，宿主侧直接复用。

一句话总结：MCP 规范的不是模型怎么思考，而是 **Agent 与外部世界之间那根线怎么接**。

## 上手：跑通最小链路

以在 OpenClaw 里挂一个本地 MCP Server 为例：

1. **选传输方式**：本地脚本、单机自动化选 stdio；跨机器、多人共享选 Streamable HTTP。
2. **实现 Server**：用官方 SDK（Python / TypeScript 都很成熟），先定义两三个工具，每个写清 name、description、参数 schema。
3. **接入宿主**：在 OpenClaw 的 MCP 配置里登记 server 命令或 URL，重启后确认工具列表被正确拉取。
4. **用 Inspector 调试**：官方 MCP Inspector 可以脱离宿主直接调用你的 server。先在这里把每个工具手动跑通，再进 Agent 链路。

一个原则：**先手动验证 server，再让模型去调**。跳过第一步，后面的排障成本会翻好几倍。

## 踩坑点

- **stdio 下污染 stdout**：`print` 调试日志会冲乱 JSON-RPC 流，连接直接断开。日志一律走 stderr。
- **工具描述写得太省**：模型选工具全靠 description 和 schema，写得含糊调用就会飘。把它当成写给新同事的 API 文档来写。
- **什么都做成 tool**：只读数据该用 Resources——要不要读配置不是模型该“决定”的事。
- **忽略超时与长任务**：耗时操作不做进度回报，宿主侧容易误判挂死。
- **安全边界想当然**：MCP Server 以你的本地权限运行，工具结果会进入模型上下文，存在提示注入面。删数据、发邮件这类破坏性操作务必加确认层和最小权限。

## 可复用建议

- 一个 Server 只管一个领域，别写大而全的“超级工具箱”。
- 工具粒度对齐动词：查、列、创建、删除，一个工具一件事，返回结构化且精简的结果。
- 优先用 SDK 和 Inspector，不要手写 JSON-RPC。
- 把 server 当独立服务维护：版本化、写测试、在主流宿主上各跑一遍兼容性。

## 总结

MCP 不是让模型变聪明的魔法，它解决的是纯工程问题：把 Agent 接入外部能力的成本从 O(M×N) 降到 O(M+N)，并把 tools、resources、prompts 三种上下文契约标准化。对 OpenClaw 社区的实践者来说，核心价值是**一次实现、处处复用**。协议仍在演进，但“先跑通最小链路、再逐步加固”的路径，现在就成立。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/8953ee7dfc7546e2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/63e3e03cfa991c27.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-17/289046013d2c8813.png)

