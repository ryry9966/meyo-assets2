---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 42731
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

主会话是 agent 的工作记忆：系统提示、长期约定、当前任务状态都压在同一个 context 里。跑长流程时，我们经常要拉子 agent 做探索性工作——扫代码、试方案、批量处理文件。这类工作天然产生大量中间产物：工具调用记录、读进来的文件内容、失败的尝试。

## 问题

如果子 agent 和主会话共用 session，或者把子 agent 的完整 transcript 回灌主会话，会出现几类麻烦：

- 中间过程塞满上下文窗口，真正的任务约定被挤出有效注意力范围，甚至触发 compaction；
- 子 agent 试错时的调用模式“传染”主会话，行为漂移；
- 探索性结论和最终决策混在一起，后续轮次说不清该信哪个；
- token 成本随子任务数量线性膨胀。

我们最早踩的是第一条：一个批量重构拉了三个子 agent，主会话被压到重排，之前注入的代码规范全丢了。

## 做法

OpenClaw 里子 agent 隔离的核心是“独立 session + 单向数据面”，落地分五步：

1. **spawn 时强制新 session**。子 agent 启动参数显式指定独立 session，不继承主会话历史，保证它的 transcript 从零开始。
2. **输入侧只给 task brief**。用固定 schema 传四样东西：目标、约束、可用工具清单、产物格式。不要把主会话历史整段贴进去——那是换一种方式污染。
3. **输出侧只回传 result**。约定子 agent 结束时只输出结构化结果：结论、关键证据（文件路径/数据行）、未解决的问题。过程叙述一律不回传。
4. **限定工具和写入范围**。tool allowlist 按任务最小化，工作区写到独立子目录（如 `workspace/subtask-xxx/`），避免和主流程文件互相覆盖。
5. **记忆写入门控 + 预算上限**。memory write 默认只允许主会话；子 agent 设 token 上限和最大步数，探索型任务跑飞就熔断。

一个可参考的 spawn 配置骨架（字段名以你实际使用的版本为准）：

```yaml
spawn:
  session: isolated
  tools: [read_file, write_file, run_tests]
  workspace: workspace/refactor-migration/
  budget: { max_tokens: 100000, max_steps: 40 }
  brief: { goal: "...", constraints: [...], output_schema: "result.v1" }
```

## 踩坑点

- **brief 塞太满**：把主会话最近二十轮对话当“背景”传下去，等于把污染从 transcript 挪到了输入。brief 超过一屏就该重写。
- **result 里夹带过程**：子 agent 习惯性写“我先做了 A 又试了 B"。在 output schema 里只定义结论字段，过程字段干脆不给。
- **共享 workspace**：两个子 agent 同时写同一个文件，谁后写谁生效，排障极其痛苦。一个子任务一个目录，是硬规矩。
- **子 session 复用**：为省启动成本反复用同一个子 session，缓存的全是上一次的脏上下文。子 session 一次性用完即弃。
- **临时结论进了长期记忆**：子 agent 的探索性判断如果没有门控，会被 memory 持久化，之后主会话把它当事实引用。

## 可复用建议

- 把“一进一出”契约固化成模板：每种子任务（代码审查、批量处理、方案验证）各维护一份 brief/result schema，团队共用。
- 主会话只保留决策和最终产物；过程性内容归档在子 session，需要追溯时按 session id 去查，而不是留在 context 里。
- 每次长流程跑完，审计主 session 的 token 构成：子任务相关内容占比超过两三成，说明回传没收紧。

## 总结

Session 隔离的本质不是“多开几个会话”，而是控制信息流向：输入给 brief，输出收 result，过程留在子 session 里自生自灭。把这条边界守住，主会话才能长期保持干净、可预测，子 agent 才是杠杆而不是负担。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/a92d14394317105c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/69ea65208be639bb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/63a935ae7c4b4946.png)

