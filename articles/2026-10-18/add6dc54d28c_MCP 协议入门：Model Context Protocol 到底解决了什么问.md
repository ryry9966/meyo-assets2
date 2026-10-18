---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 41916
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景：一个老问题的标准化尝试

做过 Agent 开发的人大概都有类似体验：模型本身不难接，难接的是工具和数据。文件系统、数据库、内部 API、浏览器自动化……每接一个新工具，都要重写一遍胶水代码——定义 schema、处理鉴权、解析错误、维护描述。接三个工具还行，接十个就开始互相打架。

这就是所谓的 **M×N 问题**：M 个应用 × N 个工具，每一对组合都要单独适配。Anthropic 在 2024 年底开源了 Model Context Protocol（MCP），目标是把它压成 **M+N**：应用实现一次 MCP client，工具实现一次 MCP server，中间走统一协议。

## MCP 到底给了你什么

MCP 本质是 JSON-RPC 之上的一层约定，核心概念不多：

- **Tools**：模型可调用的动作，比如"创建工单""执行 SQL"；
- **Resources**：可读取的上下文，比如文件内容、配置项；
- **Prompts**：预置的提示词模板。

对 Agent 框架（比如 OpenClaw）来说，接上一个 MCP server，等于批量获得一组结构化的工具定义，不用再为每个工具手写 adapter。这就是它的全部价值——不神秘，但够用。

## 动手：在 OpenClaw 里接一个 MCP server

以最常见的 filesystem server（stdio 传输）为例：

1. 确认所用的 OpenClaw 版本已启用 MCP 支持；
2. 在配置中添加 server 条目：`command`、`args`、`env`，指向一个 MCP server 可执行文件；
3. 重启网关，用工具列表相关命令确认 server 已连接、工具已注册；
4. 让 agent 实际调用一次，观察工具描述是如何被拼进上下文的——这一步对理解 token 消耗很有帮助。

远程 server 用 streamable HTTP，配置从 `command` 换成 `url` 即可，其余流程相同。

## 踩坑记录

1. **stdio 进程生命周期**。stdio 模式下 server 是网关的子进程，agent 重启它就跟着重启，长状态（如浏览器登录态）会丢。有状态的工具建议拆成独立 HTTP 服务。
2. **工具太多撑爆上下文**。一个 server 动辄暴露三四十个工具，描述全部进 prompt，token 很快吃光。按需启用子集，或一个域一个 server。
3. **工具描述质量决定调用质量**。description 含糊，模型就开始瞎猜参数。务必写清"何时该用、何时不该用、参数约束"。
4. **远程 server 的鉴权**。stdio 天然本地，HTTP server 别裸奔，至少上 token，注意 header 配置。
5. **注入风险**。工具返回内容会进入模型上下文，恶意网页或文件可能夹带指令。高危操作务必加人工确认。

## 可复用建议

- 先用官方 MCP inspector 单独调试 server，确认协议层没问题再接 agent，分层排错；
- 自研 server 时一个域一个，工具数量控制在十个以内；
- 参数 schema 写严：枚举、必填、范围校验做在 server 端，别指望模型自觉；
- 每次调用记日志：入参、出参摘要、耗时——排查时这是唯一线索；
- 写文件、执行命令、资金类操作，一律加确认门槛。

## 总结

MCP 没有黑魔法，它只是把"应用怎么调用工具"这件事标准化了。价值不在协议多先进，而在生态：别人写好的 server 你直接接，你写的 server 也能给别人用。对 OpenClaw 用户的务实路径是：先接现成 server 跑通流程，再按需自研，全程警惕上下文膨胀和注入风险。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/111dd53b629abb73.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/2209c976f36a985e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/381edcfa57172996.png)

