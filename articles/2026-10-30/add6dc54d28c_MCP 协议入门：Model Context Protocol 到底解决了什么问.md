---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42973
source: 综合讨论
publishedAt: 2026-10-30
---

## 背景：被重复造的轮子

过去一年做 Agent 插件，最耗时的往往不是模型调优，而是"接线"：接一个 GitHub 要写一套 function calling 封装，接内部数据库再写一套，换个 Agent 框架又得重写适配层。M 个应用 × N 个数据源，就是 M×N 份胶水代码。

2024 年底 Anthropic 开源了 MCP（Model Context Protocol），思路很直接：把"模型应用如何调用外部工具和数据"标准化成一个开放协议。一年多下来生态已经跑起来，值得每个做插件和自动化的人花半天搞清楚它到底解决了什么。

## 它解决的三件事

1. **接口统一**。MCP 基于 JSON-RPC 2.0，传输层用 stdio（本地进程）或 Streamable HTTP（远程服务）。工具的定义、调用、返回都有固定格式，Agent 侧的 Client 实现一次，就能对接任意数量的 MCP Server。
2. **能力发现**。Client 启动后调 `tools/list`、`resources/list` 动态获取服务端能力，不用在代码里硬编码工具清单。Server 加了新工具，Host 侧自动可见。
3. **上下文边界**。工具描述、入参 schema、返回内容如何进入模型上下文，协议层面有了约定。谁暴露能力（Server）、谁决定策略（Host），职责分开。

协议里三个核心原语值得记住：**Tools**（模型可调用的函数）、**Resources**（应用可读取的数据）、**Prompts**（预置模板）。

## 上手路径

1. 用官方 Inspector 起一个示例 Server，看懂 `initialize` → `tools/list` → `tools/call` 的完整交互。
2. 用 SDK（TS / Python 都有）写一个只读 Server，比如包一层内部 API 的查询接口，重点打磨每个工具的 description 和 JSON Schema。
3. 本地用 stdio 跑通后，再考虑用 Streamable HTTP 部署成常驻服务。
4. 接入你的 Agent（OpenClaw 的插件体系可以直接消费 MCP Server），打开工具调用日志，观察模型实际怎么选工具、传参数。

## 踩坑记录

- **description 就是 prompt**。写得含糊，模型就会错误调用或漏调用。描述"什么时候该用这个工具"比罗列参数更有用。
- **工具数量爆炸**。一个 Server 塞 30 个工具，全部描述进上下文，token 先爆，选择准确率后爆。拆成多个小 Server，或做工具过滤。
- **返回体失控**。一次查询返回几千行直接灌进上下文。默认分页、限制条数，大结果落盘后返回引用。
- **stdio 进程生命周期**。Client 异常退出后 Server 容易变僵尸进程，注意超时清理和重启逻辑。
- **写操作要防御**。模型可能重试、可能参数幻觉，非幂等操作务必加确认机制和幂等键。

## 可复用的建议

- 把 MCP Server 当 API 产品做：schema 严谨、错误信息结构化（模型能读懂并自我纠正）、调用日志带 trace id。
- 只读先行，写操作灰度放开。
- 工具粒度宁小勿大，正交组合优于大而全。

## 总结

MCP 没有让模型变聪明，它只是把管道标准化了——真正的难点仍然是接口设计和上下文工程。但对插件开发者来说，它划出了一条清晰边界：能力暴露给协议，策略留在 Host 侧。仅这条边界本身，就值得把存量胶水代码逐步迁过去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/f38f67004b27bd40.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/830eb3b2983d1bdc.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-30/d9cbaa64f454500c.png)

