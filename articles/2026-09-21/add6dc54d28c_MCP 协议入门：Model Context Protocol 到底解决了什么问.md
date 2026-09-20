---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38302
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

给模型接外部能力这件事，过去两年的常态是"各写各的胶水"：Agent 框架有自己的 plugin 格式，IDE 有自己的扩展约定，桌面助手又是一套 function calling。同一个"查内部数据库"的能力，换一个宿主就要重写一遍，彼此互不兼容。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标就是把"模型应用 ↔ 外部能力提供方"这一层标准化。可以把它理解成 AI 应用的 USB-C：宿主实现一次客户端，能力方实现一次 server，中间靠协议对话。

## 它到底解决什么问题

核心是 **M×N 集成问题**。M 个应用要接 N 种工具/数据源，没有标准时是 M×N 套适配代码；有了 MCP，收敛成 M+N。你写的 server 不绑定任何宿主，谁支持协议谁就能用。

其次它把职责切干净了，协议里三类原语各管一摊：

- **Tools**：模型可以调用的动作（发消息、跑查询）
- **Resources**：应用侧管理的上下文数据（文档、配置）
- **Prompts**：用户手动触发的模板

能力方不用关心宿主怎么渲染、模型怎么路由，只暴露契约。

## 最小可用路径

1. 选官方 SDK（Python / TypeScript 都有），先起一个 stdio 传输的本地 server
2. 用 `@mcp.tool` 装饰器暴露第一个工具，比如查内部 wiki，二三十行就能跑
3. 认真写 description 和 JSON Schema 参数——这是给模型看的接口文档，不是给人看的注释
4. 用官方 Inspector（`npx @modelcontextprotocol/inspector`）单独调试，确认工具能被列出和调用
5. 在宿主（OpenClaw、Claude Desktop、各类 IDE）的 MCP 配置里注册，跑通端到端
6. 需要团队共享时，换 Streamable HTTP 传输，前面加鉴权

## 踩坑点

1. **stdout 污染**：stdio 传输下 stdout 是协议通道，`print` 调试信息会直接打崩 JSON-RPC 流。日志一律走 stderr，这个坑几乎人人都踩过。
2. **工具描述写得太省**：description 模糊、参数没有示例，模型要么不调、要么乱传参。工具描述本质是 prompt engineering，值得投入一半工时。
3. **工具数量膨胀**：单个 server 塞 30 个工具，选择准确率肉眼可见地下滑。按领域拆 server，单 server 控制在个位数到十几个工具。
4. **写操作没有闸门**：删数据、对外发消息这类动作，协议本身不替你做权限，默认要自己加确认层。
5. **协议版本漂移**：spec 迭代很快（2024-11-05 → 2025-03-26 → 2025-06-18），宿主和 server 版本要对齐，升级前先看 changelog。
6. **Windows 下 stdio 起不来**：配置里直接写 `npx` 常常拉不起进程，需要套一层 `cmd /c`。

## 可复用建议

- MCP server 只做薄封装，业务逻辑留在已有的内部 API 里，协议层随时可替换
- 错误返回结构化、模型可读的文本，让模型能自我纠错，而不是抛一段堆栈
- Resources 和 Tools 别混用：静态数据走 Resources，动作走 Tools
- 把 server 当产品维护：版本号、变更记录、兼容性说明，一个都别少

## 总结

MCP 不是模型能力上的突破，而是一层务实的基础设施标准化：把重复的胶水代码收敛成协议，把"能力供给"从"应用开发"中解耦出来。工程量的重心从写适配器，转移到了设计清晰的工具契约上。如果你的 Agent 正在反复造查库、发通知、读文档的轮子，值得花一个下午把其中一块封装成 MCP server——接入一次，所有兼容宿主直接复用，收益很直接。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/66532b197297d170.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/0afa95db10ebea35.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/9bd54491cece14b6.png)

