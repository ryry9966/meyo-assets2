---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38813
source: 综合讨论
publishedAt: 2026-09-24
---

如果你同时维护过「多个 Agent 接多个工具」的组合，大概体会过那种痛苦：桌面端 Assistant 接一套，自己的 Agent 框架接一套，命令行脚本再接一套。同一个数据库查询逻辑，要按不同宿主的插件格式重写三遍。

MCP（Model Context Protocol）就是冲着这个问题来的。它是 2024 年底由 Anthropic 开源的标准协议，基于 JSON-RPC，定位很克制：不规定模型怎么思考，只规定「AI 应用」和「上下文提供方」之间怎么说话。社区常见的比喻是"AI 应用的 USB-C"——比喻有点营销味，但方向是对的。

## 它实际解决了什么

没有 MCP 时，工具接入是 M×N 问题：M 个宿主 × N 个工具，每对组合都要写适配代码。MCP 把它压成 M+N：工具方实现一次 Server，宿主方实现一次 Client，中间靠协议对接。

对 OpenClaw 这类 Agent 运行时来说，这意味着社区里现成的 MCP Server（数据库、浏览器、文件系统、各类 SaaS）理论上都能挂进来，不需要每个都手写插件。

协议层面只有三个原语，值得记住：

- **Tools**：模型可以主动调用的动作（如"执行 SQL"）
- **Resources**：宿主可以读取的数据（如某个文件的内容）
- **Prompts**：预置的提示词模板

实践中 90% 的时间在跟 Tools 打交道。

## 最小可用路径

1. 选官方 SDK（Python 或 TypeScript），别自己手撸 JSON-RPC。
2. 用 `@modelcontextprotocol/inspector` 起调试面板，先在宿主外把 Server 跑通——这一步能过滤掉 80% 的低级错误。
3. 定义 2~3 个工具，写清楚入参出参 schema，用 stdio 传输跑本地。
4. 在宿主配置里注册，观察实际的工具调用日志，确认模型选对了工具、传对了参数。
5. 稳定后再考虑 Streamable HTTP，部署成远程服务。

## 踩坑点

- **工具描述是写给模型看的，不是写给人看的。** 描述含糊，模型就会在该调用时不调用、不该调用时乱调用。按写 prompt 的标准写 description。
- **工具数量失控。** 挂 30 个工具后，上下文被 schema 撑爆，模型选择准确率肉眼可见地下降。按需启用，别全量加载。
- **stdio 进程生命周期。** 宿主退出会杀掉 Server 进程，别在内存里存跨会话状态，把 Server 当无状态服务设计。
- **返回给模型的错误信息。** 直接抛 stack trace 既浪费 token 又让模型胡乱重试。返回一句模型能据此行动的话，比如"表不存在，可用表为 X/Y"。
- **安全边界。** 装第三方 MCP Server 等于在本机跑别人的代码；工具返回的内容（比如抓回来的网页）可能携带注入指令。危险操作务必加确认环节。

## 可复用的建议

- 一个 Server 只做一个领域，宁可拆成多个小的。
- 工具命名保持稳定，改名等于删掉重教一遍模型。
- 每次 `tools/list`、`tools/call` 都留日志——排查"模型为什么没调工具"时，这是唯一线索。
- 能幂等的工具尽量幂等，模型重试比你想象中频繁。

## 总结

MCP 解决的是接口标准化问题，不是智能问题。它不会让你的 Agent 变聪明，但能让"接一个新工具"从半天的工程变成改几行配置。真正的价值仍然取决于工具本身设计得好不好——协议是管道，不是水。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/3d50d6502687ab6d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/2c44963634ae5d3d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/1dfa3373201ae36e.png)

