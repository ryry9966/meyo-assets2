---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 42438
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

如果你在 OpenClaw 里接过工具调用，大概率经历过这种场景：为了让 Agent 读一个本地文件、查一次数据库，要手写一套 function calling 的胶水代码；换个框架，又得重写一遍。MCP（Model Context Protocol）就是冲着这个来的——Anthropic 在 2024 年底开源的协议，目标是把「模型怎么连工具和数据」这件事标准化。

## 它到底解决什么问题

一句话：**把 M×N 的集成问题变成 M+N。**

- M 个 AI 应用 × N 个数据源/工具，以前要写 M×N 个适配器；
- 有了 MCP，工具方实现一次 MCP Server，应用方实现一次 MCP Client，双方按同一份协议通信。

MCP 采用 client-server 架构，消息层是 JSON-RPC 2.0，传输支持 stdio（本地子进程）和 Streamable HTTP（远程）。Server 对外暴露三类东西：

1. **Tools**：由模型决定何时调用的函数，带 JSON Schema 入参描述；
2. **Resources**：由应用控制的数据，比如文件内容、查询结果；
3. **Prompts**：面向用户的调用模板。

## 做法：五步跑通最小闭环

1. 选一个支持 MCP 的宿主（Claude Desktop、主流 IDE 插件，或你自己的 Agent 框架）；
2. 先配置一个现成 Server（filesystem、git 之类）验证链路，stdio 模式下就是一段 JSON 配置；
3. 用官方 SDK（Python / TypeScript）写自定义 Server，用装饰器声明 tool，认真填入参 schema；
4. 用 MCP Inspector 调试，它会列出 Server 暴露的所有 tools/resources，可以直接手动调用验证；
5. 上线：本地走 stdio，远程走 Streamable HTTP，记得配鉴权。

## 踩坑点

- **stdout 千万别污染**。stdio 模式下 stdout 就是协议通道，一句 `print` 调试日志就能把连接打挂，日志一律走 stderr；
- **工具描述就是 prompt**。模型靠 description 选工具，写得含糊就会被误调用。别写「处理数据」，要写「读取指定路径的 CSV，返回前 N 行的列统计」；
- **工具数量失控**。一口气挂 50 个工具，上下文先膨胀，选择准确率也会掉。按场景拆分 Server，按需启用；
- **协议在演进**。2025-03 版规范用 Streamable HTTP 取代了 HTTP+SSE，老教程的传输配置可能直接跑不通，SDK 注意锁版本；
- **别盲信第三方 Server**。工具返回内容会进入模型上下文，本质上是一条注入通道。文件系统权限收窄到工作目录，不认识的 Server 别给写权限。

## 可复用建议

- 一个 Server 只管一类事，别做上帝模块；
- 把 tool description 当 prompt 写，写完自问一句：模型看了这个描述，能选对吗？
- 返回值尽量结构化，长输出做截断或分页，别把 10MB 日志塞给模型；
- 长耗时工具做好超时与进度反馈；
- 把自己写的 Server 当独立组件维护，README 写清权限需求和使用边界。

## 总结

MCP 不会让你的 Agent 变聪明，它做的是把「接工具」这件脏活的边际成本打下来。对 OpenClaw 这类自动化实践者，价值在于：工具层可以独立开发、独立复用、跨宿主迁移。建议路径很简单——先拿官方 Server 验证链路，再把业务能力封装成自己的 Server。先跑通最小闭环，再谈架构。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/3e4a7c18b118e5fa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/2a4ceb6151c8e2be.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/d14ad480030b4661.png)

