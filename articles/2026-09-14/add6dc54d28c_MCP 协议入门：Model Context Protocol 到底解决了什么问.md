---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 37565
source: 综合讨论
publishedAt: 2026-09-14
---

# MCP 协议入门：Model Context Protocol 到底解决了什么问题

## 背景

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），此后一年多，它基本成了 Agent 生态里连接外部工具的事实标准：桌面客户端、各类 IDE 插件、开源 Agent 框架陆续接入，社区 server 数量快速膨胀。但对刚接触的人来说，MCP 的定位很容易和 Function Calling、OpenAPI 混在一起，说不清它到底比现有方案多解决了什么。

## 问题：M×N 的集成困境

MCP 出现之前，如果要让 Agent 操作 GitHub、查数据库、读本地文件，你得为每个「Agent 框架 × 工具」的组合单独写一遍对接：

- 框架有 M 个，工具/数据源有 N 个，集成成本就是 M×N
- 每个框架定义工具的方式不同：schema 格式、调用方式、错误处理各搞一套
- 工具提供方不可能为每个框架维护一份官方适配
- 本地文件、内网服务这类资源，模型没有统一的访问通道

关键区别在于：Function Calling 解决的是「模型怎么调函数」，但没有规定函数从哪来、怎么描述、怎么发现、怎么鉴权。MCP 补的就是这一层。

## 做法：核心设计 + 上手路径

MCP 本质是基于 JSON-RPC 2.0 的协议，client-server 架构：

- **Host**：运行模型的应用（Agent、IDE、桌面客户端）
- **Client**：Host 内部与某个 Server 保持 1:1 连接的中间层
- **Server**：暴露具体能力的轻量服务，比如一个封装了 PostgreSQL 查询的进程

Server 对外暴露三类原语：**Tools**（模型主动调用的执行类操作）、**Resources**（应用读取的只读数据）、**Prompts**（用户触发的提示词模板）。传输层常见两种：本地进程走 stdio，远程服务走 Streamable HTTP。

建议的上手路径：

1. 先装一个现成 server（filesystem、git）在本地跑通，观察 Host 和 Server 之间的实际报文
2. 用官方 SDK（Python/TypeScript）写一个最小 server，只暴露一个带清晰 description 的 tool
3. 挂到自己的 Agent 里，验证工具发现、调用、错误回传三个环节
4. 确认稳定后，再逐步替换项目里手写的 Function Calling 封装

## 踩坑点

- **工具数量失控**。挂到 30 个 tool 之后模型选择明显变差，description 之间互相干扰。按领域拆 server，按需加载。
- **description 写得敷衍**。模型完全靠 description 决定调不调、怎么调，参数说明含糊会直接导致错误传参。
- **返回结果太大**。整张表、整段 JSON 直接回传会迅速吃掉上下文，过滤和分页必须在 server 侧做完。
- **stdio 进程环境问题**。工作目录、环境变量、写死的绝对路径，是最常见的翻车点。
- **安全别大意**。tool 返回内容会进入模型上下文，等于打开了注入面；凭证放环境变量，敏感操作加确认环节。

## 可复用建议

- 能用现成 server 就别自己写，官方和社区维护的 filesystem、git、数据库 server 覆盖了大部分场景
- 写 description 的验收标准：一个不了解你系统的初级工程师，只看描述就能正确传参
- 一个 server 只管一个领域，出错定位快，也方便独立开关和灰度
- server 侧记录所有调用日志——排查 Agent 行为异常时，这是唯一可靠的线索

## 总结

MCP 没有发明新魔法，它做的是把「模型如何连接工具」标准化：统一的描述格式、发现机制和传输方式，把 M×N 的集成成本压到 M+N。对 OpenClaw 这类自动化实践场景，直接收益是工具层可以独立开发、复用和替换。建议先跑通官方示例建立直觉，再考虑迁移现有逻辑——协议本身不复杂，真正需要花心思的是你工具边界的划分。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/2e4d5bc41fe0d60c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/d210766044355af6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/a0983c5fef941e35.png)

