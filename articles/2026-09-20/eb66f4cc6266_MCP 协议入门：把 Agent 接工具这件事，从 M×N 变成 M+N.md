---
title: MCP 协议入门：把 Agent 接工具这件事，从 M×N 变成 M+N
feedId: 38223
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

过去一年做 Agent 自动化，最耗时间的往往不是模型，而是"接线"：让 Agent 能查数据库、调内部 API、操作文件系统。每个宿主（OpenClaw、各类 IDE 助手、桌面客户端）都有一套自己的插件格式，同一个"查工单"能力，换个宿主就得重写一遍接入层。M（个宿主）× N（个工具）的集成成本，就是这么堆出来的。

## MCP 到底解决了什么问题

Model Context Protocol（MCP）是一个开放协议，2024 年底发布，核心思路很朴素：把"模型如何发现并调用外部能力"这件事标准化。它定义了三样东西：

- **传输与消息格式**：基于 JSON-RPC 2.0，本地走 stdio（宿主拉起子进程），远程走 Streamable HTTP；
- **能力原语**：tools（供模型调用的函数）、resources（可读取的上下文数据）、prompts（可复用的提示模板）；
- **发现机制**：客户端连接后通过 `tools/list` 拿到工具清单和参数 schema，不需要硬编码。

效果是把 M×N 压缩成 M+N：工具方只写一次 MCP Server，宿主方只实现一次 Client，双方按协议对接。工具用什么语言写、跑在哪里，宿主不关心。

## 上手步骤

以在 OpenClaw 里接一个最简单的 MCP Server 为例：

1. **确认传输方式**。本地工具选 stdio，宿主以子进程方式拉起 server；远程服务选 Streamable HTTP，配 URL 即可。
2. **在宿主配置里注册 server**。stdio 方式需要给出启动命令、参数和环境变量（如 `command: npx`、`args: [...]`）；HTTP 方式给 url 和鉴权头。
3. **重载后验证发现**。确认宿主正确列出工具、参数 schema 完整。这一步通了，后面基本只剩调参。
4. **真实调用一轮**。让 Agent 实际跑一次工具调用，检查入参解析、返回内容和超时表现。

## 踩坑点

- **工具描述决定调用成功率**。模型选工具完全依赖 name/description/schema。我们有个内部工具因 description 写得太抽象，调用成功率不到一半，重写描述后失败基本归零。
- **stdio 环境差异**。npx 在 Windows 下需要 `cmd /c` 包一层；server 依赖的 PATH、HOME 与宿主进程不同，会出现"手动能跑、宿主拉不起来"。
- **上下文膨胀**。挂十几个 server、上百个工具，工具清单本身就吃掉一大块上下文，还推高选错工具的概率。按需启用，别全挂上。
- **命名冲突**。多个 server 暴露同名工具时，注意宿主的前缀/隔离策略。
- **安全别省**。MCP Server 等于把文件系统和网络交给 Agent。第三方 server 先审再装，用最小权限账户跑，能沙箱就沙箱。

## 可复用建议

- **工具粒度宁粗勿细**：一个"创建工单"比 create/validate/submit 三个原语好用得多，模型规划链路越短越稳。
- **返回值要短、要结构化**：大段原始 JSON 塞回上下文，费 token 还干扰判断，先在 server 侧做裁剪和摘要。
- **长任务拆开**：改成"提交任务 + 查询状态"两个工具，避免单次调用阻塞到超时。
- **沉淀配置模板**：把验证过的 server 配置做成团队模板，新人接入成本能降到分钟级。

## 总结

MCP 没有魔法，它只是把 Agent 接工具从"各写各的胶水代码"变成"按协议对接"。对实践者的价值很直接：工具写一次，处处能挂；宿主换一个，配置照搬。协议本身仍在演进，但方向已经清楚——如果你的自动化链路里还有第三段一次性集成代码，值得评估换成 MCP。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/11e5f56c82c9ca73.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/3e4f5d91faa46a2e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/28c8b14fc51a7599.png)

