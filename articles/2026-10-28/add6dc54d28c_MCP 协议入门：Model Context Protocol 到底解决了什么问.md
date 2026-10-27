---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42783
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

Agent 开发里有个绕不开的现实：模型本身只会生成文本，所有“干活”的能力——读文件、查数据库、调外部 API——都得靠外挂的工具调用。在 MCP 出现之前，这件事没有统一标准。

每个客户端（IDE、桌面助手、Agent 框架）自定义一套 function calling 格式，每个工具提供方要为不同客户端分别适配。假设有 M 个客户端、N 个工具，最坏要做 M×N 套集成，而且互不通用。你在自己 Agent 里写好的工具封装，换个宿主基本要重写一遍。

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），目标是把 M×N 压缩成 M+N：客户端只实现一次协议，工具只封装一次 server，双方按标准对接。一年多下来，它已经是 Agent 生态里事实上的工具接入标准。

## MCP 到底定义了什么

MCP 本质是一套基于 JSON-RPC 2.0 的协议，架构分三层：Host（你的 Agent 应用）、Client（Host 内部与 server 保持连接的实例）、Server（暴露能力的进程）。Server 对外提供三类核心原语：

- **Tools**：可被模型调用的操作，如“创建 issue”“执行 SQL”；
- **Resources**：可被读取的数据，如文件内容、日志片段；
- **Prompts**：预置的提示词模板，用于固定工作流。

传输层主流两种：本地进程用 stdio，远程服务用 Streamable HTTP。协议还覆盖了能力协商、进度通知、请求取消等细节——这些恰恰是自研集成最容易漏掉的部分。

## 上手步骤

以最小可跑为例（Python SDK）：

1. `pip install "mcp[cli]"`，写一个 20 行左右的 server，用 `@mcp.tool()` 暴露一个查询函数；
2. 用官方 inspector 自测：`mcp dev server.py`，确认工具 schema 和返回结构符合预期；
3. 在 Agent 宿主里以 stdio 方式注册该 server（配置启动命令即可），重启后验证模型能否正确选中工具并传参；
4. 需要团队共享时，把 server 改为 Streamable HTTP 部署，补上鉴权再暴露出去。

## 踩坑点

- **stdout 污染**：stdio 模式下 stdout 是协议通道，任何 `print` 日志都会导致连接解析失败，日志必须写 stderr；
- **工具描述即接口**：模型靠 description 和 JSON Schema 决定是否调用、怎么传参，描述含糊就会“该调不调、传错参数”；
- **工具数量膨胀**：几十个 tool 全量塞进上下文，既挤占 token 又降低选择准确率，按需挂载或分组加载；
- **协议版本差异**：早期远程 server 用的 SSE 传输与新版 Streamable HTTP 不完全兼容，接第三方前先确认；
- **安全别裸奔**：server 能直接触达文件系统和网络，工具返回内容也可能夹带注入指令，自动放行所有 tool call 是高危配置。

## 可复用建议

- 一个 server 只管一个领域；粒度宁可粗一点，合并相关操作，减少模型决策次数；
- 工具返回值做裁剪，错误信息是写给模型看的，不是写给运维看的；
- 把 inspector 当默认调试器，先在协议层验证通过，再接进宿主排查；
- server 带版本号和变更记录，工具行为变更要显式同步给所有接入方。

## 总结

MCP 解决的不是“模型变聪明”，而是“能力接入标准化”。它把散落在各个 Agent 项目里的私有集成，收敛成一个可复用、可分发的协议层。对 OpenClaw 社区的实践者来说，判断标准很简单：如果某段工具集成逻辑你已经写了两遍以上，就值得封装成一个 MCP server 了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/323bd42466fcfc71.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/5b6a92ff651b0d7d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/fb18ba160403aead.png)

