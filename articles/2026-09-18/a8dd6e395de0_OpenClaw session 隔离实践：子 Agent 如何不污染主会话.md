---
title: OpenClaw session 隔离实践：子 Agent 如何不污染主会话
feedId: 38037
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

在 OpenClaw 的编排模型里，主会话是唯一对用户负责的上下文：它承载对话历史、长期记忆和任务状态。跑自动化时，检索、批量执行、代码运行这类子任务通常派给子 Agent。子 Agent 有独立 session，但"有自己的 session"不等于"不会污染"，边界要靠编排层主动维护。

## 问题：污染的三种形态

1. **转写污染**。子 Agent 的中间推理和工具原始输出被整段 append 进主会话 history，几千 token 的调试过程挤占主上下文，主 Agent 注意力被稀释，后续回答质量明显下降。
2. **记忆污染**。子 Agent 持有主会话的 memory 句柄，把任务中的临时结论写进长期记忆，下一次完全无关的对话里被错误的"经验"带偏。
3. **状态污染**。并发子 Agent 同时写同一个文件或同一个键，互相覆盖，出现只在新会话复现的诡异 bug。

## 做法

我们的实践收敛成五步：

1. **派生而非共享**。用 `session_spawn` 创建子 session，父子只靠 `parent_id` 关联。子 session 的初始上下文是手工编译的 handoff——任务目标、硬约束、输入数据路径——而不是主 history 全量。
2. **结果走显式通道**。约定 result schema，子 Agent 结束时只返回一个结构化对象，主会话只 merge 它。原始转写留在子 session 内，任务结束即归档：

```json
{ "status": "ok", "summary": "…", "artifacts": ["/tmp/task-42/out.csv"], "idempotency_key": "task-42" }
```

3. **记忆分域**。子 Agent 只能写 session 级 scratchpad；要沉淀为长期记忆，必须由主会话确认结果后显式写入 workspace memory。
4. **资源隔离**。每个子 session 使用独立的临时工作目录和工具白名单；MCP 工具注册时声明写作用域，编排器据此拒绝越界写入。
5. **生命周期收口**。`session_close` 或 TTL 到期自动清理；编排层必须处理超时和失败回执，不允许静默挂起。

## 踩坑点

- **handoff 塞"以防万一"的全量上下文**：token 爆炸是小事，把主会话里的敏感内容带给子任务才是事故。最小上下文是纪律，不是优化。
- **忘记 close**：孤儿 session 累积，占配额、持文件锁，往往 dump 了 session 树才发现。
- **失败盲目重试**：子 Agent 超时后主会话直接重派，写文件、发通知这类副作用会重复执行。result 里放幂等键，重试前先查。
- **隔离过头**：子 Agent 拿不到必要凭据或约束，产出不可用，白跑一轮。最小上下文不等于零上下文，边界要靠试跑校准。

## 可复用建议

- 把 handoff 模板和 result schema 版本化管理，当成接口维护，改动走 review。
- 定期 dump session 树做审计，重点看"谁在写主记忆"。
- 给子 session 设 token 预算，超限尽早失败，比在主会话里爆掉便宜得多。
- 新子任务类型上线前，先在沙箱主会话跑一次"污染演练"，验证清理逻辑真的生效。

## 总结

隔离的本质是**子 Agent 只带回结论，不带回过程**。session 是边界，schema 是契约，生命周期是兜底。把这三件事做实，多 Agent 编排才不会随着任务数增长变成一锅粥。欢迎在社区帖子里附上你们的 session 树 dump 和失败案例，比空谈架构有用得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/aba10ba4c46c09e0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/d02aac9c8986dd38.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/6d3928684804cbbe.png)

