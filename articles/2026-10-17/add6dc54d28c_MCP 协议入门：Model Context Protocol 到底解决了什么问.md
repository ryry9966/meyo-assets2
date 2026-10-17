---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 41806
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

过去一年做 Agent 落地的人大概都经历过类似的场景：模型能力够了，卡在工具层。想让 Agent 查数据库、调内部 API、操作文件系统，每接一个模型、每换一个框架，工具接入代码都得重写一遍。Function calling 解决了“模型怎么表达要调工具”，但没解决“工具怎么以统一方式提供给模型”。这就是经典的 N×M 问题：N 个应用 × M 个数据源，每个组合都要单独写胶水代码。

## 问题

具体拆开，痛点有三个：

1. **集成重复**。同一个“查工单库”的能力，在这个客户端里写一遍，在自己的 Agent 框架里又写一遍，接口、鉴权、错误格式全都对不上。
2. **上下文供给没有约定**。工具调用之外“把数据塞给模型”这件事，各家做法五花八门，文件、检索结果、配置的暴露方式不统一。
3. **关注点耦合**。工具的实现、部署、鉴权和模型侧代码缠在一起，模型一换，工具层跟着返工。

## 做法

MCP（Model Context Protocol）用 JSON-RPC 2.0 作为消息层，把这一层标准化了。核心角色只有三个：

- **Host**：Agent 本体（桌面客户端、自研框架），内嵌 MCP Client；
- **Server**：能力提供方，按域拆分，比如一个管数据库、一个管文件；
- **Transport**：本地用 stdio，远程用 Streamable HTTP。

Server 对外暴露三类原语：**Tools**（可执行操作，有副作用）、**Resources**（只读数据）、**Prompts**（预置模板）。

实操下来，接入是五步：

1. 用官方 SDK（Python / TypeScript）起一个最小 Server，先只暴露一个工具；
2. 给每个工具写清楚 name、description、JSON Schema——这三样是模型唯一能看到的东西；
3. 用官方 Inspector 单测 Server，不接模型先验证调用链路；
4. 选传输方式：本地进程用 stdio，跨机器用 HTTP 加鉴权；
5. 在 Host 侧接通：启动时拉工具列表 → 注入模型 → 模型返回调用请求 → Client 转发 Server → 结果回填上下文。

跑通一个之后，第二个、第三个 Server 基本就是复制结构的事。

## 踩坑点

几个代价比较高的坑：

- **Schema 含糊，模型就乱调**。description 不是给人看的注释，是给模型的指令。`query_data` 不如 `Query the orders table by date range; returns at most 50 rows`。
- **大返回值打爆上下文**。查库返回 200 行 JSON，一次调用吃光 token 预算。解法是 Server 侧做分页和摘要，或返回 ID 让模型按需取。
- **stdio 模式的环境问题**。Server 依赖的 Python 环境、npx 版本和 Host 假设不一致，症状是“连不上”，实际是启动就崩。先在终端手动跑一遍 Server 启动命令。
- **把 Server 输出当可信内容**。工具返回的网页、工单文本里可能藏注入指令，Host 侧要做隔离审查，别直接拼进系统提示词。
- **规范在演进**。早期 SSE 传输已换成 Streamable HTTP，SDK 版本间行为有差异，锁版本、看 changelog。
- **带副作用的工具没有确认层就是事故**。删数据、发消息类操作，Host 必须加人工确认或白名单。

## 可复用建议

- 把每个工具当成写给初级工程师的接口说明：name 说做什么，description 说什么时候用，schema 约束怎么传参。
- 读写分离：只读查询走 Resources 思路，变更走 Tools，权限和审计分开做。
- 一个 Server 管一个域，别做上帝 Server；鉴权、部署、日志都跟着域走。
- 所有调用落日志：工具名、参数、耗时、返回大小，排障时全靠它。
- 先 Inspector 验证，再接模型，最后上自动化，不要跳步。

## 总结

MCP 不是什么智能增强，它是管道标准——把“模型怎么找工具、怎么调工具、怎么拿结果”固定下来。它的价值不体现在单次集成提速多少，而在于工具层从此可以独立于模型和框架演进，生态里写好的 Server 能直接复用。对做 Agent 自动化的人来说，与其再手写一套胶水层，不如把精力花在工具的 Schema 和返回值设计上——协议解决的是通信，好用与否仍取决于接口设计本身。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/289c6ef6e9d2d89c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/cfe7e27b09c78fca.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/a5311bad3ca1e8bb.png)

