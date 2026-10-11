---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 41209
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景

每个做 Agent 的人早期都手写过一遍 function calling：系统提示里塞工具定义、解析模型输出、自己执行、把结果拼回上下文。Demo 阶段没问题，但当你有 3 个 Agent 应用、10 个数据源（文件系统、数据库、内部 API、工单系统），就要维护 30 份胶水代码，换个模型厂商还得重写一遍工具协议。这就是经典的 N×M 集成问题。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标就是把这个问题降成 N+M：工具方实现一次 MCP Server，任何支持 MCP 的 Host（Agent 运行时、IDE、桌面客户端）都能直接接入。

## 它到底标准化了什么

MCP 基于 JSON-RPC 2.0，核心做了三件事：

1. **能力发现**：Host 通过 `tools/list` 拿到服务器暴露的工具清单，每个工具带 JSON Schema，模型据此决定何时调用。
2. **统一调用**：`tools/call` 走同一套报文格式，Host 不关心工具背后是 shell 脚本还是 SaaS API。
3. **进程/网络边界**：本地服务器走 stdio，独立进程运行，崩了不连累主程序；远程服务器走 Streamable HTTP，凭证可以集中管理。

协议还定义了 resources（只读数据）和 prompts（模板），但实际 80% 的场景只用 tools 就够。

## 上手步骤

建议先跑现成服务器，再自己写：

1. 启动官方 filesystem 服务器（stdio 传输）：
   ```bash
   npx @modelcontextprotocol/server-filesystem ~/projects
   ```
2. 在客户端配置里声明这个 server（OpenClaw 的 MCP 配置块或 `claude_desktop_config.json`），写清命令、参数、允许访问的目录。
3. 重启后验证发现是否生效：问一句"你能访问哪些文件"，或用 `@modelcontextprotocol/inspector` 可视化调试，能直接看到 `tools/list` 和 `tools/call` 的完整报文。
4. 自己写服务器时用官方 TS/Python SDK，核心就是给每个工具定义 name、description、inputSchema，再实现对应 handler。

## 踩坑点

- **description 是写给模型看的**。"查询数据"这种描述模型根本不会用；要写清楚什么场景该调用、参数格式、失败时返回什么。
- **stdio 服务器别往 stdout 打日志**。stdout 被 JSON-RPC 占用，日志混进去会直接搞挂会话，日志一律走 stderr。
- **工具数量爆炸**。每个工具的 schema 都吃上下文 token，挂 40 个工具后模型的选择质量明显下降。按场景拆分 server、按需启用。
- **安全别裸奔**。filesystem 服务器授权到根目录、DB 服务器用可写账号，都是真实出过事的配置。默认最小权限 + 路径白名单。
- **警惕第三方 server 的结果注入**。工具返回内容会进模型上下文，恶意页面可能借工具结果诱导下一步高危调用，这类操作务必保留人工确认。

## 可复用建议

- 一个 server 只做一类事，小而稳比大而全好维护。
- 大结果集做分页或摘要，别一次性灌进上下文。
- 上线前用 inspector 把每个工具的异常路径跑一遍，确认错误信息模型能看懂。
- 优先复用社区现成 server，接入前确认维护活跃度和权限模型。

## 总结

MCP 解决的是工程问题，不是模型能力问题：把"每个应用适配每个工具"的集成矩阵，变成一次实现、处处接入的协议层。它不会让 Agent 变聪明，也不会替你解决鉴权和可观测性的全部细节，但能把工具生态的接入成本压到接近边际为零——这正是 Agent 从 demo 走向生产最缺的那块基础设施。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/22b4d6dc7b630e9b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/6f19192f44d9eab4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/85bef965ee5caf61.png)

