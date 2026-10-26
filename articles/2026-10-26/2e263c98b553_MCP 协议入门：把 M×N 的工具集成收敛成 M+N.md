---
title: MCP 协议入门：把 M×N 的工具集成收敛成 M+N
feedId: 42647
source: 综合讨论
publishedAt: 2026-10-26
---

## 背景：M×N 的集成地狱

写过 Agent 或自动化脚本的人大多经历过：接一个新数据源，就要重写一遍胶水代码——鉴权、分页、错误处理、结果格式化，再把工具的 JSON Schema 手工拼进 system prompt。换个框架、换个模型，流程重来一遍。工具接多了之后，维护成本是 M×N：M 个应用 × N 个工具。

MCP（Model Context Protocol）解决的就是这个问题。它是 Anthropic 2024 年底开源的协议，一句话概括：给“模型应用”和“外部工具/数据”之间定一个标准插座。底层是 JSON-RPC 2.0，核心只有三个原语：

- **Tools**：模型可调用的动作，有副作用；
- **Resources**：应用可读取的上下文，只读；
- **Prompts**：预置的提示词模板。

配套还有能力协商（initialize 握手）、工具发现（tools/list）和两种传输层（本地 stdio / 远程 Streamable HTTP）。工具 schema 不用自己拼 prompt，客户端在会话初始化时自动拉取注入。

## 做法：三步跑通第一条链路

1. **选客户端**：Claude Desktop 或任意支持 MCP 的框架；自研 Agent 用官方 TypeScript/Python SDK 接入，OpenClaw 用户走插件接入的思路同理。
2. **起一个现成 server**：`npx @modelcontextprotocol/server-filesystem /path/to/dir`，一行命令起一个文件读写 server，配置里声明 command 和 args 即可。
3. **验证**：重启客户端，确认工具已出现，然后让它“列出目录下所有 md 文件并总结”。想看协议细节，用官方 Inspector（`npx @modelcontextprotocol/inspector`）直接观察握手报文和每次调用的原始 JSON。

## 踩坑点

- **工具描述就是半个 prompt**。模型选工具、填参数全靠 description 和参数说明。写成函数名级别的含糊描述，调用成功率直接砍半。
- **工具数量吃上下文**。每个 schema 都会注入上下文，十几个 server、几十个工具，光定义就可能占几万 token。宁少勿多，粒度合并。
- **stdio 只适合本地**。远程部署走 Streamable HTTP 时，鉴权必须自己做（OAuth 或网关层），别裸奔。
- **工具返回值是不可信输入**。它直接进模型上下文，等于开了注入面。读外部数据的工具要做截断和消毒，有副作用的工具加确认机制。
- **按 capabilities 降级**。不同 server 支持的原语和特性不同，别假设所有 server 都有 Resources。

## 可复用建议

- 内部系统优先封装成 MCP server，而不是在 Agent 里堆 if-else 胶水——写一次，所有支持 MCP 的客户端都能复用，这是它相对自定义 function calling 最大的红利。
- 读数据用 Resources，做动作用 Tools，语义分开，权限才好收。
- 把 schema 当 prompt 写：参数给例子、写清边界条件和失败时的行为。
- 上线前过一遍 Inspector 报文，确认没有把敏感字段暴露给模型。

## 总结

MCP 不是让模型变强的魔法，它解决的是工程问题：把“应用接工具”从 M×N 的私有集成，收敛成 M+N 的标准对接。协议本身很薄，真正的功夫在工具设计——描述怎么写、粒度怎么切、权限怎么收。这和当年设计 REST API 是同一类问题，经验可以直接迁移。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/71f2b9bfbfc751b5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/3af30bf492cad7e4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-26/5f18f566ca398437.png)

