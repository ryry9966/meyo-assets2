---
title: MCP 协议入门：把 M×N 的集成问题变成 M+N
feedId: 40496
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的一套协议，目标很朴素：给模型和外部工具、数据源之间定义一个统一接口。一年多过去，主流客户端和 Agent 框架基本都支持了，社区里 Server 数量也起来了。如果你在做 OpenClaw 插件或自动化流水线，绕不开一个决策：工具接入是自己封装，还是走 MCP。

## 它到底解决了什么问题

核心是组合爆炸：**M×N → M+N**。

- N 个工具/数据源（数据库、GitHub、内部 API、文档系统……）
- M 个模型宿主 / Agent 运行时

没有标准协议之前，每接一个工具，就要为每个宿主写一遍 function calling 封装、鉴权和错误处理；工具换个接口，M 个地方一起改。MCP 把工具侧统一成 Server，宿主侧只实现一次 Client，两边按协议对接，集成成本从相乘变成相加。

顺带还解决了三个实际问题：

1. **能力声明标准化**：Server 通过 JSON Schema 自描述有哪些 tools/resources，宿主动态发现，不用硬编码。
2. **上下文边界**：数据留在各自的 Server 里，模型按需调用，而不是把所有数据都塞进 system prompt。
3. **凭证收口**：鉴权配置在 Server 一侧，模型拿到的只是调用入口，不直接持有密钥。

## 动手：最小可跑的接入路径

以最常见的场景为例——让你的 Agent 能查本地 SQLite：

1. **选传输方式**：本地工具用 stdio（宿主拉起子进程），远程服务用 streamable HTTP。
2. **用官方 SDK 写 Server**（Python / TypeScript 都有），实现一个 tool，声明名字、描述和入参 Schema。描述写得好坏，直接决定模型的调用准确率。
3. **在宿主端注册**：把这个 Server 配置进 OpenClaw 或任意 MCP 客户端，重启会话。
4. **验证**：先确认 tools list 拉取成功，再跑一次真实调用，观察入参是否被模型正确填充。

一个体会：第 2 步里 tool 描述和参数命名，值得花的时间不比业务代码少。

## 踩坑点

- **工具列表撑爆上下文**：一口气挂十几个 Server，上百个 tool 的 schema 全进系统提示，token 消耗和选择准确率都会崩。按需启停，别贪多。
- **stdio 进程生命周期**：宿主退出时子进程没被正确回收，文件锁和连接残留。做幂等设计，升级传输层时注意会话状态。
- **第三方 Server 的安全**：MCP Server 能读文件、能执行命令。装别人发布的 Server 之前看一眼源码，至少弄清它申请了什么权限，别把整个 home 目录挂进去。
- **Schema 质量**：参数描述含糊，模型就会瞎猜传参。枚举值尽量用 enum，别指望在 description 里写「请传某种格式」。
- **协议仍在演进**：spec 出现过不兼容调整（如 SSE 改为 streamable HTTP），客户端和 Server 的 SDK 版本要对齐。

## 可复用建议

- 工具只被一个宿主用，直接写原生插件更快；要跨宿主、跨项目复用，才值得包成 MCP Server。
- 一个 Server 聚焦一个数据域，工具粒度对齐「一次可验证的动作」，不要把整套 CRUD 都暴露出去。
- 上线前做三件事：核对 tools list、跑通一次真实调用、确认凭证没出现在模型可见的上下文里。

## 总结

MCP 没有黑魔法，它只是把「模型怎么调外部工具」这件事协议化了。价值在于生态复用和边界清晰，代价是多一层抽象，加上一个仍在演进的协议本身。判断标准很简单：工具要被多个宿主共享，用 MCP；只在单点内部使用，不必为了标准而标准。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/058762f788a83fd7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/127fa6921ac37620.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/c8dd465f15b05b31.png)

