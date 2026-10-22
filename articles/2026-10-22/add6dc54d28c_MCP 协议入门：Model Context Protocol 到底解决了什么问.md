---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42250
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

Agent 能力上限越来越取决于它能"够到"什么。早些时候给 Agent 接工具基本是点对点硬写：接 GitHub 要写一份 connector，接数据库再写一份，换个 Agent 框架，同样的活再干一遍。工具数量 × Agent 框架数量，集成就成了 N×M 的重复劳动，而且各家接口格式互不兼容。

## MCP 解决了什么

Model Context Protocol 的核心思路很简单：把"Agent 怎么调外部能力"标准化成一个开放协议。它统一的不只是工具调用，还包括上下文的注入方式。

协议里三个角色：Host（Agent 应用本体）、Client（Host 内维护连接的组件）、Server（暴露能力的轻量服务）。对开发者来说最关心的是 Server 暴露的三类原语：

- **Tools**：可执行的操作，模型主动调用（查数据库、发消息、跑脚本）
- **Resources**：可读取的数据，类似只读文件（配置、文档、日志）
- **Prompts**：预置的提示模板，用户侧触发

一个 MCP Server 写一次，任何兼容协议的 Host 都能用——这就是把 N×M 降成 N+M 的方式，可以类比为 AI 应用的 USB-C。

## 上手路径

1. **先跑现成的**。社区有 filesystem、fetch、git 等常用 Server，在 Host 配置里加一段 JSON（command + args + env），重启就能看到工具被加载。
2. **理解传输层**。本地跑用 stdio（进程直连），远程用 Streamable HTTP。日常实验 stdio 最省事。
3. **写一个自己的 Server**。用官方 SDK（Python/TypeScript）二三十行就能包一个内部 API：声明 tool 的 name、description、JSON Schema 入参，注册 handler 即可。
4. **验证调用链**。让 Agent 实际调一次，观察参数是否按 Schema 生成，这是排查问题的第一步。

## 踩坑点

- **description 就是给模型看的接口文档**。写得太含糊，模型要么不调、要么传错参。把"什么时候用 / 什么时候不用 / 返回什么"写清楚，准确率提升立竿见影。
- **工具不是越多越好**。一次挂 30+ 个工具，上下文变胖，选错工具的概率明显上升。按场景分组配置，用到哪组挂哪组。
- **权限要收敛**。MCP Server 跑在你本机权限下，filesystem Server 把整个家目录暴露出去等于裸奔。用 roots 限定目录，只开必要的 allowlist。
- **工具结果也可能有注入**。Server 返回内容会进模型上下文，别默认它可信，敏感操作前保留人工确认。
- **Windows 下 stdio 编码**。中文环境常见 GBK/UTF-8 冲突导致 Server 起不来，显式设置 `PYTHONUTF8=1` 或改走 HTTP 传输可绕开。

## 可复用建议

- 优先复用社区 Server，只有包内部 API 时才自己写
- 一个 tool 对应一个清晰动作，别做万能大接口
- MCP 配置纳入版本库，不同项目挂不同 Server 集合
- 给 Agent 开工具调用日志，出问题能分清是 Server 的锅还是模型选型的锅

## 总结

MCP 没有魔法，它做的是把一个存在多年的集成问题标准化：工具提供方只写一次 Server，Agent 使用方只实现一次 Client。对 OpenClaw 用户来说，理解 Tools / Resources / Prompts 三类原语和权限边界，基本就能把 MCP 用在正确的位置。它解决的是"接入层"问题，别指望它替你设计工具粒度和安全模型——那部分仍然是工程师的活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/29646bece7538921.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/20932ea68623ab0b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/8796bfae5efe5b74.png)

