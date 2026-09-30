---
title: Agent 与 API 的握手：OpenClaw 对接外部服务的工程化做法
feedId: 39859
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

OpenClaw 本身不生产能力，它更像一个调度器：对话进来，模型决定“做什么”，真正“做事”的是你接进来的外部服务——内部 API、第三方 SaaS、本地脚本。握手质量直接决定 agent 是“能用”还是“敢用”。

社区里常见的三种接法：

1. **Skill 脚本**：写个 shell/Python 脚本放进 skills 目录，agent 通过工具调用执行；
2. **MCP Server**：把 HTTP API 封装成 MCP 工具，schema 显式、权限可控；
3. **网关直连**：适合只读、低风险的查询类接口，先跑通再考虑收编成工具。

## 问题

实际对接时，坑基本集中在四件事：

- 模型不知道你的 API 长什么样，参数靠猜；
- 鉴权信息放哪——放进 prompt 等于裸奔；
- 接口返回一大坨 JSON，上下文瞬间被撑爆；
- 网络抖动一重试，订单下了两单。

## 做法

**第一步：先用 MCP 封装，别让模型直接拿 curl。** 把外部 API 包一层 MCP tool，每个工具一个窄接口，JSON Schema 里的 enum、required、description 全写满。schema 就是给模型看的接口文档，写得越死，幻觉越少。

**第二步：鉴权走环境变量。** API key 放 env 或 secrets 配置，工具运行时注入，绝不进对话历史，也绝不打印进日志。

**第三步：返回值裁剪。** 工具层做字段过滤和分页，只返回模型决策需要的字段。一个经验阈值：返回给模型的 JSON 超过几百 token，就该在工具里先砍一刀。

**第四步：错误要“说人话”。** 超时、4xx、5xx 不要原样抛堆栈，封装成一句模型能读懂并据此行动的描述，比如“参数 date 格式应为 YYYY-MM-DD，请修正后重试”。错误信息本质上是写给模型的提示词。

**第五步：写操作必须幂等。** 带 client 生成的 request id，重试时复用同一个 id；高风险操作加 dry-run 参数和人工确认环节。

## 踩坑点

- **schema 太松**：参数全是 string，模型就会自由发挥。能用 enum 就 enum，能限长度就限长度。
- **长任务阻塞**：同步等 60 秒的接口会卡死会话，改成“提交任务 + 轮询查询”两个工具。
- **接口漂移**：上游改了字段名，agent 不报错但悄悄返回空。给每个工具配一个最小 e2e 测试，放进 CI。
- **secret 进日志**：工具调用日志记得脱敏，key 前后各留几位即可。

## 可复用建议

- 工具粒度宁小勿大：`create_ticket` 是好工具，`manage_workspace` 是灾难。
- 每个工具先手写三条典型调用的 golden case，验证通过再交给 agent。
- 用一份 YAML 描述 endpoint、超时、重试策略，工具代码读配置生成，避免十几个工具各写一套。
- 上线前用只读账号灰度，把工具调用日志接进现有观测栈，别只靠翻对话记录。

## 总结

对接外部服务本质上是两份契约：一份是给模型的 schema，一份是给运维的配置。前者决定 agent 会不会用，后者决定你半夜会不会被叫起来。把工具当产品来做——窄接口、强 schema、可幂等、可观测——OpenClaw 才能从“能对话”走到“能交付”。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/32321b18ca299f96.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/ed26a19a58d5286b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/9536711f691efb94.png)

