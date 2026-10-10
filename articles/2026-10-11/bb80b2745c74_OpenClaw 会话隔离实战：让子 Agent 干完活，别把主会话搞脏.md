---
title: OpenClaw 会话隔离实战：让子 Agent 干完活，别把主会话搞脏
feedId: 41147
source: 综合讨论
publishedAt: 2026-10-11
---

## 背景

用 OpenClaw 编排多 Agent 做自动化任务时，常见结构是：主会话里的主 Agent 负责拆解任务、调度子 Agent（或通过 MCP 调外部工具），子 Agent 负责具体执行。早期我们没有严格区分会话边界——子 Agent 的工具输出、中间推理、报错日志直接回流主会话上下文。任务一多，主会话很快被填满。

## 问题

污染主要体现在三个层面：

1. **上下文膨胀**：一次网页抓取的原始 HTML、一段日志分析的全部行，都进了主会话，token 成本和响应质量同时恶化。
2. **行为漂移**：主 Agent 会"记住"子 Agent 的中间结论，后续决策被无关细节带偏，比如把某次失败重试的临时方案当成既定事实。
3. **状态串扰**：多个子 Agent 共享同一份工作目录和记忆命名空间时，互相覆盖文件、互相"继承"中间变量，排查起来非常痛苦。

## 做法

OpenClaw 目前的思路是：**主会话只做控制面，子 Agent 自带临时会话，返回值走契约**。落地分四步：

**1. 子 Agent 用独立临时会话。** 在派生配置里关闭上下文继承，默认不带入主会话历史：

```yaml
subagent:
  session: ephemeral
  inherit_context: none
  return:
    format: json_schema
    max_tokens: 800
  workspace: workspace/{task_id}/
```

**2. 定义返回契约。** 子 Agent 不回传原始过程，只回传结构化结果：结论、产物路径、关键引用。给返回值设 token 上限，超限让子 Agent 自己先做摘要。

**3. 产物落盘，路径引用。** 抓取的页面、生成的报告写到按 task_id 隔离的目录，主会话只拿路径，需要时再定点读取，而不是把内容塞进上下文。

**4. 显式提升共享记忆。** 子 Agent 默认写自己的记忆命名空间；只有主 Agent 判断"这个结论值得沉淀"，才显式写入共享记忆并打上来源标签。

验证方式：用 session inspector 检查主会话上下文构成，跑一个含三个子任务的工作流，确认主会话里只出现任务描述和三段结构化返回。

## 踩坑点

- **返回值太啰嗦**：不给 max_tokens，子 Agent 会把"摘要"写成小作文，上限一开始就要定。
- **过度隔离**：`inherit_context: none` 之后子 Agent 缺必要背景，产出跑偏。正确姿势是显式传入最小必要上下文（任务目标、约束、相关文件路径），而不是放开全部继承。
- **人设串了**：子 Agent 继承了主 Agent 的 system prompt，口吻和职责混乱，临时会话里要覆写 persona。
- **产物路径冲突**：并行子任务写同一目录互相覆盖，务必用 task_id 做命名空间。
- **临时会话泄漏**：ephemeral 会话有 TTL，但异常退出时不会清理，长跑任务要定期巡检。
- **并发写共享记忆**：多个子 Agent 同时提升记忆会互相踩，收口到主 Agent 单点写入。

## 可复用建议

- 把子 Agent 的返回当 API 设计：有 schema、有预算、可幂等重试。
- 上下文流向默认拒绝、白名单放行，而不是默认全通再补救。
- 记录会话血缘（parent_id / task_id），出问题时能沿链路回放。
- 主会话保持"决策 + 摘要"密度，原始数据一律走产物文件。

## 总结

会话隔离本质上不是开一个开关，而是接口纪律：进什么、出什么、留什么，都要有明确约定。OpenClaw 的临时会话 + 返回契约 + 显式记忆提升这套组合，让我们一条跑了两周的多 Agent 流水线把主会话 token 占用压到原来的三分之一，主 Agent 的决策稳定性也明显回升。如果你的编排开始"越长越笨"，先检查子 Agent 的会话边界。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/0829961d9c24f924.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/e54d1b25fabd73a7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-11/eb48e405f781dc60.png)

