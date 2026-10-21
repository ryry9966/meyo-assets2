---
title: 从 M×N 到 M+N：MCP 协议到底解决了什么问题
feedId: 42236
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），一年左右的时间里它基本成了 Agent 接外部工具的事实标准：编辑器、桌面客户端、各类 Agent 框架（包括 OpenClaw）都提供了 MCP 接入点。很多人的第一反应是：Function Calling 不是早就有了吗？MCP 究竟多解决了什么？

## 问题：M×N 的集成爆炸

Function Calling 解决的是"模型怎么调用工具"，但没规定"工具怎么接入某个宿主"。在 MCP 之前：

- 你写一个 Agent 框架，要为每个数据源（文件系统、数据库、内部 API、SaaS）各写一套连接器；
- 你维护一个内部系统，想被多个 Agent 使用，就得给每个宿主各写一个适配器。

M 个应用 × N 个数据源 = M×N 个集成，而且鉴权、传输、工具发现、生命周期全是各写各的。这就是 MCP 要消掉的核心成本：把 M×N 变成 M+N——工具方只需实现一次 MCP Server，宿主方只需实现一次 MCP Client，两边按协议对接即可。

## 做法：最小可用路径

MCP 本质是一套基于 JSON-RPC 2.0 的客户端-服务端协议。Server 对外暴露三类能力：tools（模型可调用的动作）、resources（可读取的上下文数据）、prompts（预置提示词模板）。本地进程走 stdio，远程服务走 Streamable HTTP（早期版本为 HTTP+SSE）。

给想上手的同学一条最小路径：

1. 用官方 SDK（TypeScript/Python）起一个最简 Server，先只暴露 2~3 个只读工具，比如"列出某目录下的文件""按 ID 查订单"；
2. 在 OpenClaw 的 MCP 配置里注册这个 Server，确认工具列表能被发现，手工触发一次真实调用；
3. 补齐错误处理和日志，再逐步加写操作，并把写操作放到人工确认环节之后；
4. 需要跨机器共享时，再从 stdio 迁到 Streamable HTTP，补上鉴权与部署。

先跑通一条链路，再谈扩展，顺序不要反。

## 踩坑点

- **工具描述是给模型看的 API 文档。** 命名含糊、描述空泛，模型就会选错工具或编造参数。描述要写清"何时该用我、参数是什么格式"。
- **stdio 模式下别往 stdout 打日志。** stdout 是 JSON-RPC 通道，混入一行日志整条流就废了，日志统一走 stderr。
- **返回内容要克制。** 把一整张表塞给模型只会烧掉上下文窗口，优先返回摘要和 ID，让模型按需再查。
- **错误信息要让模型能读。** 返回"订单不存在，请检查 ID 格式"远比一个堆栈或空字符串有用。
- **工具数量不是越多越好。** 注册几十个工具后选择准确率明显下降，按业务域拆分 Server、合并同义工具。
- **别盲目自动放行。** 工具描述和返回内容都可能被注入恶意指令，写操作务必走确认，凭证按 Server 隔离。

## 可复用建议

- 把工具 schema 当合同：写不依赖模型的集成测试，直接调用 Server 验证行为；
- 一个域一个 Server，不要造大而全的"万能 Server"；
- 先只读、后写入，写操作永远过确认；
- 锁定 SDK 与协议版本，MCP 还在演进，升级前先看 changelog。

## 总结

MCP 解决的不是"模型能不能调工具"，而是"工具生态如何避免重复造轮子"。它把集成的固定成本摊薄成一次实现、处处复用。但它只是协议：工具描述的质量、返回内容的设计、权限边界的划定，这些工程细节仍然决定你 Agent 的实际表现。协议负责标准化，效果依然靠人。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/7a54427e2103e922.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/2cbeb4e9036ee395.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/46e8ef3faa40d90b.png)

