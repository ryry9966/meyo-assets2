---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 39290
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

Agent 本身不会调 API。模型只会生成文本，真正把"帮我查一下订单状态"变成一次 HTTPS 请求的，是外层的工具层。在 OpenClaw 的体系里，这一层主要由 MCP server 和插件承接。所谓"握手"，其实就两件事：让 Agent 知道有哪些工具可用（能力发现），让每次调用能稳定跑完并拿到可读的结果（执行闭环）。

## 问题

常见的错误对接方式有三类：

1. 把 API 文档整段塞进 system prompt，指望模型自己"想办法"；
2. 直接给 Agent 一个万能 `curl` 工具，让它现拼 URL；
3. 每个服务各写一套一次性脚本，鉴权、重试、日志各搞各的。

前两种会导致调用不可控、密钥容易混进上下文；第三种则维护成本失控。核心矛盾在于：模型需要小而清晰的工具面，而真实 API 往往又大又糙。

## 做法

推荐按这个顺序走：

**1. 用 MCP 做协议层。** 把外部服务包一层薄薄的 MCP server（stdio 或 streamable HTTP 均可），每个核心端点对应一个 tool。在 OpenClaw 配置里注册 server 后，会走 `initialize` → `tools/list` 完成发现，Agent 侧看到的就是一组带 JSON Schema 的函数。

**2. 工具设计做减法。** 不要一个 tool 塞 20 个参数。按业务动词拆：`get_order`、`create_ticket`、`list_pending`。description 要写"什么时候该用我"，而不只是"我是什么"。

**3. 鉴权放在 Agent 之外。** 密钥走环境变量或本地配置，MCP server 启动时读取。对话上下文里永远不应该出现明文 token。

**4. 返回结果做裁剪。** API 的原始 JSON 先在 server 层过滤：只保留模型做决策需要的字段，错误码翻译成人话——"订单号不存在"，而不是一坨 stack trace。

**5. 慢操作走异步。** 超过十几秒的任务，tool 立即返回任务 ID，再配一个 `check_status` 工具让 Agent 轮询，避免调用被超时掐断。

## 踩坑点

- **description 含糊**：Agent 要么不调，要么乱调。写完工具自己先扮演用户跑几轮。
- **大 JSON 直接回流**：上下文膨胀、费用上涨，还可能触发截断。server 层裁剪是刚需。
- **吞掉 HTTP 错误**：失败时返回空结果，模型会一本正经地编造成功。错误也要结构化返回。
- **写操作没有幂等键**：Agent 重试一次，订单创建两份。所有 create 类工具都应支持客户端传入 idempotency key。
- **stdio server 静默挂掉**：表现为"Agent 说没有这个工具"。先翻 OpenClaw 日志里 server 的启动输出，多半是路径或依赖问题。

## 可复用建议

- 把"薄 MCP server + 密钥在环境变量 + 裁剪后的返回"当作对接模板，新服务照抄结构；
- 工具粒度对齐业务动词，一个工具一个意图；
- 给每次 tool call 记日志（入参、出参、耗时），排障时你会感谢自己；
- MCP server 用包管理器固定版本，避免上游升级悄悄改了行为。

## 总结

对接外部服务的本质，不是"让 Agent 能上网"，而是给它一组边界清晰、结果可读、失败可解释的工具。OpenClaw 侧把 MCP 协议层用稳，服务侧把鉴权和数据裁剪收进 server 层，这次握手才算真正握上。剩下的，才是 Agent 自由发挥的空间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/cb83d4986dcc99d7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/677ef9e2fd65e86b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/54421bdbf247797b.png)

