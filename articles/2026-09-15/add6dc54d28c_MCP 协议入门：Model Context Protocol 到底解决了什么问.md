---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 37651
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

LLM 应用落地到 2024 年底，瓶颈逐渐从“模型够不够聪明”转移到“Agent 接不接得上外部世界”：读数据库、调内部 API、操作本地文件、抓公司 wiki。而当时每个宿主应用，各家 IDE 助手、聊天客户端、Agent 框架——都自定义了一套插件/工具格式；一个数据源想被 LLM 用上，就得给每个宿主单独写一遍集成。

## 问题：M×N 的胶水代码

这个结构的代价很直接：M 个宿主 × N 个数据源 = M×N 份不可复用的胶水代码。给 A 客户端写的内部 API 插件，换到 B 框架就得重写。另一层问题是上下文注入全靠手工：把表结构、日志、文档内容复制粘贴进 prompt，无法程序化、无法标准化。工具的调用格式、发现机制、鉴权方式，全都没有共识。

## MCP 的答案

MCP（Model Context Protocol）是 Anthropic 在 2024 年 11 月开源的协议，基于 JSON-RPC 2.0。架构分三层角色：Host（LLM 应用）、Client（宿主内维护的连接）、Server（提供能力的独立进程）。核心只有三类原语：

- **Tools**：模型可主动调用的函数；
- **Resources**：应用侧可读取的上下文数据（文件、记录、schema）；
- **Prompts**：可复用的提示词模板。

传输上本地走 stdio，远程走 Streamable HTTP。它做的事一句话概括：**把 M×N 压成 M+N**——数据源写一次 server，所有支持协议的宿主都能复用。

## 上手步骤

1. 用官方 Python 或 TypeScript SDK 起一个最小 server，先只定义一个只读工具，比如“查询某表行数”。
2. 认真写 name、description 和 inputSchema——description 本质是写给模型看的 API 文档。
3. 用 MCP Inspector（`npx @modelcontextprotocol/inspector`）本地调试，确认 `tools/list` 和 `tools/call` 正常。
4. 挂到宿主：OpenClaw 可以把 MCP server 直接挂成 Agent 工具，其他标准 MCP 客户端也能复用同一个 server。
5. 观察模型的真实调用行为，持续迭代描述文本和参数设计。

## 踩坑点

- **stdio 模式下，任何 print 到 stdout 的内容都会污染协议流。** 日志必须走 stderr，这是新手最常翻车的地方。
- 工具描述含糊，模型要么不敢调、要么乱传参数。描述不是给人看的注释，是 prompt 的一部分。
- 工具返回大段原始数据会直接灌进上下文，token 成本爆炸。server 侧先过滤、分页、给摘要。
- 挂多个 server 时注意工具重名，确认宿主的命名空间/前缀策略。
- 远程鉴权（OAuth 相关规范）仍在演进，传输层也从早期 HTTP+SSE 改成了 Streamable HTTP，老教程代码经常跑不通，部署时跟 spec 版本走。

## 可复用建议

- 先 MCP 化“查询类、只读”能力，风险最小、收益最直接。
- 一个 server 对应一个领域，别做万能大杂烩，出问题难定位。
- 把工具返回当上下文预算管理：默认给摘要，用参数显式控制详细级别。
- 已有 REST API 不必重写，写个几十行的薄封装 server 即可接入。

## 总结

MCP 解决的是**接口标准化**，不是能力问题。它让“工具”从宿主绑定变成可移植资产，Agent 生态第一次有了类似 USB 的通用插口。但工具好不好用，仍取决于 server 作者：描述是否清晰、返回是否克制、领域切分是否合理。协议把桌子摆好了，菜还得自己做。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/84b3f4041cc25927.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/9e9debfc001ed635.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/f84233a8a72d70ad.png)

