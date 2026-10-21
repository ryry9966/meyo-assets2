---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42160
source: 综合讨论
publishedAt: 2026-10-21
---

## 背景

2024 年底 Anthropic 开源了 Model Context Protocol（MCP），一年多时间它成了 Agent 工具生态的事实标准。但社区里对它的理解常常停留在“又一种 function calling 写法”。这篇从工程视角梳理：MCP 到底解决了什么，以及在 OpenClaw 里怎么把它用稳。

## 问题：M×N 的集成地狱

没有 MCP 之前，工具接入是典型的 M×N 问题：M 个 Agent 客户端、N 个工具/数据源，任意组合都要写一份定制胶水代码。你在 OpenClaw 里接一个数据库要写一套，换个客户端再写一套；工具方想支持多个 Agent，就得维护多份适配。MCP 把它压成 M+N：工具方只实现一次 MCP server，客户端只实现一次 MCP client，中间用统一协议通信。类比就是 LSP 之于编辑器、USB-C 之于外设。

## MCP 是什么

MCP 是基于 JSON-RPC 2.0 的开放协议，三个角色：Host（如 OpenClaw）、Client（Host 为每个 server 维持一条连接）、Server（暴露具体能力）。协议里只有三类原语：

- **Tools**：模型可主动调用的动作；
- **Resources**：可读取的上下文数据；
- **Prompts**：预置的提示模板。

传输层主流两种：本地进程用 stdio，远程服务用 Streamable HTTP。连接建立时先做 initialize 握手，协商版本与能力。

## 实践步骤

以在 OpenClaw 里接入一个内部系统为例：

1. 明确需求，决定用现成 server 还是自建。官方与社区参考实现很多；内部 API 建议用 SDK 包一层自建。
2. 在 OpenClaw 配置中声明 server：本地写启动命令和环境变量，远程写 URL；用工具白名单控制暴露范围。
3. 重启后先验证工具列表被正确注入，再跑一次真实调用，核对参数回传是否完整。
4. 观察日志：调用耗时、失败率、token 占用。

## 踩坑点

1. **工具描述是写给模型看的。** 含糊的 description 会直接导致乱传参，值得花时间打磨。
2. **工具数量爆炸。** 几十个工具全部注入，上下文膨胀且选择准确率下降，应按场景启用子集。
3. **安全边界。** server 返回的内容会进入模型上下文，本质是一条注入通道；不可信的远程 server 慎接，敏感数据源放本地 stdio。
4. **stdio server 崩溃的表现是“工具突然消失”**，而不是报错。排障先看 server 的 stderr，再查客户端连接日志。
5. **版本协商失败**常见于老客户端，握手阶段就会断开，先确认双方 capability。

## 可复用建议

- 工具粒度对齐业务动作，一个工具做一件事；错误信息结构化且可读，让模型能自我纠正。
- 只读工具默认开放，有副作用的写操作加确认环节。
- 与其往 system prompt 里塞接口文档，不如把内部系统包成 MCP server，可靠得多。

## 总结

MCP 解决的不是“让模型更聪明”，而是集成的标准化问题：把一次性胶水代码变成可复用的基础设施。对 OpenClaw 用户来说，衡量一个 MCP server 的标准很朴素——描述清楚、粒度合理、失败可恢复。做到这三点，工具链才真正可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/8408e03f807d1210.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/dda968fc186acc36.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/2d105b3b140d6658.png)

