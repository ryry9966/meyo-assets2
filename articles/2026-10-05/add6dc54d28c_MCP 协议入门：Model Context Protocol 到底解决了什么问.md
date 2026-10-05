---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 40573
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

接 agent 做自动化，最耗时间的从来不是模型，而是"接线"：让 agent 能查数据库、读工单、跑脚本。每个 host（桌面客户端、IDE、自研 agent 框架）都有自己的插件格式，每个数据源都要单独包一层胶水。3 个 agent × 8 个工具 = 24 份互不通用的一次性代码，改一处全要重测。

MCP（Model Context Protocol）就是冲着这个来的。它把"agent 如何发现工具、调用工具、获取上下文"定义成一套开放协议，2024 年底开源，主流 host 和官方 SDK（Python / TypeScript）都已跟进，OpenClaw 的工具生态也在往这条路上收敛。

## 它具体解决什么

一句话：把 M×N 的集成问题降成 M+N。

- 以前：M 个 agent 各自适配 N 个工具，工具方要写 M 种封装，agent 方要写 N 种适配。
- 现在：工具方实现一次 MCP Server，agent 方实现一次 MCP Client，两边按协议握手即可。

协议里最常用的三类能力：

- **Tools**：可执行动作（查询、发消息、跑命令），入参用 JSON Schema 描述；
- **Resources**：只读上下文（文件、配置、表结构），模型可"看"不可"改"；
- **Prompts**：服务端预置的提示模板。

底层消息是 JSON-RPC 2.0，传输层本地走 stdio，远程走 Streamable HTTP。

## 最小上手路径

1. 确认你的 host 支持 MCP（OpenClaw、常见桌面客户端、IDE 均可），或用官方 SDK 自建 Client；
2. 用 SDK 写一个最小 Server，先只暴露一个 Tool，比如 `query_orders(start_date, end_date)`，把 description 写清楚；
3. 在 host 配置里注册：本地命令行工具填启动命令（stdio），远程服务填 URL；
4. 先调 `tools/list` 确认工具发现正常，再手动 `tools/call` 一次，检查返回结构；
5. 跑通后再加第二个、第三个工具，只读数据逐步拆成 Resources。

## 踩坑点

- **description 就是给模型看的 API 文档**。写得含糊，模型会在相近工具间乱选。写清"什么时候不该用我"，比罗列参数更有用。
- **工具一多，选择准确率明显下降**。别把 50 个工具全注册进去，按任务域分组或动态开关。
- **stdio Server 挂了经常无声无息**。handler 里写阻塞调用会冻住事件循环，重 IO 交给异步或子进程；stderr 日志要留着看。
- **报错别直接抛异常**。模型需要可读的错误信息才能自我修正，返回结构化 error 字段，下一轮它就能改对参数。
- **大结果会撑爆上下文**。查询类工具务必分页或截断，别把整张表吐给模型。
- **安全边界比想象中薄**。本地 stdio Server 继承你的全部用户权限，别让"读网页"和"删文件"待在同一个不经确认的 Server 里；工具返回内容属于不可信输入，防注入要自己兜底。

## 可复用建议

- 一个 Server 只管一个域：文件、Git、数据库分开，便于独立启停和控权限；
- 只读能力用 Resources，有副作用的才用 Tools，模型对"该不该确认"的判断会准很多；
- 固定 Server 版本，维护一份自己验证过的清单，社区的你没跑过的别随手装；
- 给所有 `tools/call` 留审计日志，出问题能回放。

## 总结

MCP 不负责让工具变好用，它只负责让工具可插拔。真正的收益在你积累三五个 Server 之后才显现：换 host 不用重写集成，新 agent 接入成本趋近于配置文件里的一行。建议从一个 20 行的最小 Server 起步，把 description 当 prompt 写——剩下的，都是工程细节。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/f2517902b3a9aa8d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/2064b7081cd56598.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/d69c5e1f80de203a.png)

