---
title: MCP 协议入门：把 M×N 的集成问题变成 M+N
feedId: 37532
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

2024 年底 Anthropic 开放了 Model Context Protocol（MCP），一年多过去，它基本成了 Agent 生态里"工具接入"的事实标准，OpenClaw 的工具/插件体系也在向它靠拢。但社区里常见两种极端看法：一种把它当银弹，一种觉得"不就是 function calling 换个壳"。这篇不站队，只讲清楚它到底解决了什么工程问题。

## 它解决的问题：M×N 的集成地狱

MCP 之前，给 Agent 接一个外部能力（查数据库、读文件、调内部 API）的路径是：宿主定义一套 function calling 格式 → 为每个工具写适配代码 → 工具一改就重新发版。

如果你有 M 个 Agent 应用、N 个工具，最坏要维护 M×N 份胶水代码，而且各家框架的 schema 格式互不兼容。

MCP 把这件事收敛成一个协议：工具方只实现一次 MCP server，任何支持 MCP 的宿主（Agent 框架、IDE、OpenClaw 这类助手运行时）都能发现并调用它，**M×N 变成 M+N**。宿主在运行时通过 JSON-RPC 动态发现工具（`tools/list`），新增工具不需要改宿主代码。

## 协议最小集：三个角色，两类传输

- **Host**：Agent 运行时，持有模型与对话上下文；
- **Server**：能力提供方，暴露 Tools（模型可调用的函数）、Resources（可读数据）、Prompts（模板）；
- **传输**：本地走 stdio（宿主拉起子进程），远程走 Streamable HTTP。

一个最小 Python server 只要十几行：`@mcp.tool()` 装饰器加一个带类型标注的函数，SDK 自动生成 JSON Schema。建议的实践路径：

1. 用官方 SDK 写一个只含**一个工具**的 server；
2. 用 MCP Inspector 单独调试，确认 schema 和返回结构；
3. 接入宿主时先只暴露这一个工具，验证完整调用链路；
4. 稳定后再逐步加工具。

## 踩坑点

1. **工具描述就是 prompt 的一部分**。模型靠 name + description 决定调不调、怎么调。描述含糊，选错工具的概率直线上升。写描述的口吻应该是"给新来的实习生写使用说明"，而不是 API 文档。
2. **stdio 的环境问题**。宿主拉起子进程时的 PATH、解释器版本和你终端里的不一定一致，command 写绝对路径、钉死版本，是新手翻车率最高的一类问题。
3. **工具数量爆炸**。一次挂 50 个工具，选择准确率明显下降。按领域拆 server，按场景开关。
4. **返回内容别裸奔**。把 10MB JSON 直接塞回上下文会瞬间打爆窗口。server 侧做分页、过滤、摘要，返回里带 ID 供后续查询。
5. **安全边界**。MCP server 是持凭证的可信代码，工具返回会进入模型上下文——恶意 server 可以借返回内容诱导模型外带数据（tool poisoning）。社区 server 先读代码再装，文件系统访问收到最小范围。
6. **幂等性**。模型可能重复调用同一工具，写操作要么天然幂等，要么加确认门。

## 可复用建议

- 一个 server 只管一个领域，宁可多拆几个；
- 每次工具调用都落日志（入参、出参、耗时），排查"模型为什么乱调"时这是唯一可靠证据；
- 准备一组固定测试 prompt 做回归：改了描述就跑一遍，看工具选择是否漂移；
- 破坏性操作统一走"先预览、后确认"模式。

## 总结

MCP 没有让模型变聪明，它只是把"接入"从 N 份私有胶水代码收敛成一个公开协议，把工程重心从管道挪到了工具设计本身。对 OpenClaw 用户来说，务实的态度是：把它当成一个稳定的插座标准来用，同时在描述质量、返回结构、安全边界上花真功夫——这部分协议帮不了你，只能自己做好。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/c8447955a5f32f6b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/51c6dceb64d91544.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/76bf048c94248ce7.png)

