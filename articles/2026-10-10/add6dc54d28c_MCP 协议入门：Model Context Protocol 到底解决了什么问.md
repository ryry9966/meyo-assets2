---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 41071
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

接工具这件事，过去两年基本是“一事一议”：Agent 想调 GitHub，就手写一套 function calling 封装；换到本地文件、数据库、内部 API，再各写一遍。宿主侧同样碎片化——不同客户端的工具格式、传输方式、鉴权约定互不兼容。MCP（Model Context Protocol）做的事情，是把“模型应用 ↔ 工具/数据源”这一层抽成一个公开协议：基于 JSON-RPC 2.0，定义 tools / resources / prompts 三类能力，支持 stdio 和 Streamable HTTP 两种传输。类比的话，它想当 AI 接入外部世界的 USB-C。

## 它解决的真实问题

核心是 M×N 集成问题：M 个宿主 × N 个工具，以前要写 M×N 份胶水代码；有了 MCP，工具方写一次 server，任何兼容宿主都能发现并调用。除此之外还有三个容易被忽略的点：

- **动态发现**：宿主启动时向 server 拉取工具列表和 schema，不用硬编码；
- **职责分离**：鉴权、数据访问收敛在 server 内，宿主只管编排；
- **生态复用**：文件系统、Git、浏览器这类通用 server 已经有人写好，多数场景不必从零造。

## 上手步骤（最小可用路径）

1. 选一个支持 MCP 的宿主（OpenClaw、桌面客户端或自研 Agent 框架均可）；
2. 用官方 SDK（Python/TypeScript）写一个最简 server，只暴露一个工具，比如 `list_files`，把描述和 JSON Schema 写清楚；
3. 用 stdio 本地跑通，确认宿主能发现并正确调用该工具；
4. 验证通过后再按领域逐步加工具；远程部署时再切 Streamable HTTP 并补鉴权。

先窄后宽。一个 server 真正跑通，比一上来铺十个工具有价值得多。

## 踩坑点

- **工具描述是写给模型看的，不是 API 文档**。描述含糊，模型就会选错工具或编造参数，这是新手问题里占比最高的一类；
- **工具数量失控**。几十个工具全注册，上下文被塞爆、选择准确率明显下降，该过滤就过滤；
- **传输方式选错场景**。stdio 只适合本地子进程，远程服务别硬套；
- **安全别想当然**。server 以你的权限运行，工具返回内容可能携带注入指令，敏感操作务必加确认机制和最小权限；
- **协议版本在演进**。客户端与 server 的规范版本不匹配时，常出现莫名的握手失败，排查前先对版本。

## 可复用建议

- 按 Agent 意图设计工具（如 `search_issues`），而不是照搬底层原始 API；
- 返回结果做裁剪，别把大段 JSON 直接灌进上下文；
- 长任务提供进度或异步返回，避免调用超时；
- 每次工具调用留日志，出了问题能回放定位。

## 总结

MCP 不会让 Agent 变聪明，它解决的是工程问题：把接入成本从 M×N 压到 M+N，让工具生态可组合、可复用。对 OpenClaw 用户来说，把常用能力封装成几个职责清晰、描述准确的 MCP server，是当前性价比最高的自动化基建投入。有落地经验欢迎在社区里回帖交流。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/d471793db7f618ff.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/dee90103d35318f1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/76093a49656225a1.png)

