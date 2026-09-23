---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 38694
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景：工具集成的 M×N 问题

写 Agent 的人大多经历过这样的循环：想让它读 GitHub issue，写一份 function calling 适配；想查数据库，再写一份；换个模型或框架，格式不同，胶水代码重写一遍。N 个应用乘 M 个数据源，集成成本是乘法关系。MCP（Model Context Protocol）就是针对这个乘法问题提出的开放协议——把"模型如何发现并调用外部能力"标准化，把 M×N 变成 M+N。

## MCP 到底是什么

一句话概括：MCP 定义了 Host（Agent 运行时）、Client、Server 三方的通信规范。Host 内嵌 Client，通过统一协议连接一个或多个 MCP Server；每个 Server 封装一类能力，对外暴露三种原语：

- **Tools**：可被模型调用的操作，如创建 PR、执行查询
- **Resources**：可被读取的数据，如文件内容、表结构
- **Prompts**：预置的提示模板

传输层支持 stdio（本地子进程）和 streamable HTTP（远程服务）。OpenClaw 这类运行时扮演 Host 角色，接入成本集中在配置 Client 和挑选 Server 上。

## 接入步骤（以本地 stdio 为例）

1. **选 Server**：从官方和社区仓库里挑，优先选择维护活跃、提供只读模式的。
2. **注册配置**：在 OpenClaw 的 MCP 配置段声明 server 启动命令、参数、环境变量，并设置权限白名单。
3. **启动握手**：拉起子进程，确认 initialize 握手成功，列出 tools 和 resources。
4. **冒烟测试**：用一条明确指令触发一次调用，核对入参出参是否符合 schema。
5. **收敛权限**：确认默认拒绝写操作，敏感目录和凭证不要进环境变量。

远程场景只需把第 2 步换成 endpoint 加鉴权配置，其余流程一致。

## 踩坑记录

- **工具描述质量决定调用准确率**。Schema 里 description 含糊，模型就会乱传参或选错工具。自己写 server 时，把参数约束写进 description 和 enum，比事后改 prompt 有效得多。
- **工具不是越多越好**。挂 30 个工具，上下文占用膨胀，模型的选择开始漂移。按任务场景分组启用，而不是全量挂载。
- **stdio 的进程生命周期**。Server 随 Host 启停，调试时它的 stderr 常常混进主日志，定位问题前先分离日志流。
- **版本兼容**。协议在演进，Client 与 Server 版本差距大时，轻则能力静默降级，重则握手失败。升级后务必重跑冒烟测试。
- **权限治理**。一个带写权限的 MCP Server 约等于交出一部分 shell，生产环境坚持最小权限加调用审计。

## 可复用建议

- 接入顺序：先 resources（只读），后 tools（可写），逐步放开。
- 每个 server 配一份最小冒烟脚本，纳入 CI 或升级检查清单。
- 多 server 并存时给工具名加前缀，避免同名冲突。
- 记录每次 tool call 的入参、出参、耗时——排查"模型为什么调错"时全靠这份日志。

## 总结

MCP 解决的不是"模型不够聪明"，而是集成的工程问题：统一了能力发现、描述与调用的协议层。它最大的收益是复用——工具写一次，任何兼容的 Host 都能挂载。但协议本身不负责权限边界和 server 质量，这两块仍是接入方自己的功课。对 OpenClaw 用户的务实路径是：从一两个只读 server 开始跑通全链路，验证日志与权限行为，再逐步扩展到写操作和远程部署。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/286f6e89b571384c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/29b3ffa8283f0f4b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/f722184b38634ddd.png)

