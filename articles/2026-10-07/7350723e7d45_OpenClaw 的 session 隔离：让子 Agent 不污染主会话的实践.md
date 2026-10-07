---
title: OpenClaw 的 session 隔离：让子 Agent 不污染主会话的实践
feedId: 40816
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

OpenClaw 的主 agent 是一个长会话：session 文件持续追加，配合滚动摘要压缩上下文，历史越攒越多。跑自动化流程时，我们经常需要派子 agent 去干"脏活"——批量读代码、检索资料、试跑命令。这类任务会产生大量中间工具结果，天然就不该进主会话。

## 问题

早期我们在插件里直接复用主会话对象派生子 agent，踩过不少坑：

- **上下文膨胀**：三个子 agent 并行跑完，主会话凭空多出几万 token 的工具噪声，滚动摘要被迫提前触发；
- **记忆污染**：子 agent 调 memory 写入，把中间猜测当结论写进了长期记忆；
- **时间线错乱**：子 agent 的消息出现在主对话记录里，事后回放审计对不上账；
- **文件竞争**：多个子 agent 同时追加同一个 session JSONL，偶发行交错。

## 做法

现在团队内的约定是"一次任务、一个会话、一份摘要回来"，对应五个动作：

1. 子 agent 一律用 fresh session，不继承历史，只注入任务描述和必要的局部上下文（相关文件路径、约束条件）；
2. 回传只取 final message，且要求结构化：结论、变更文件列表、未解决问题。中间推理和工具输出全部留在子会话文件里；
3. 工具面收窄：默认只给只读工具，需要写就写 scratch 工作区（按 task id 命名），由主 agent 决定是否合并；
4. MCP 按 session 挂载：有状态的 server 在子 agent 侧用独立 namespace，或标记为 main-only；
5. 设上限：max turns + 超时，防止子 agent 失控重试烧 token。

配置示意（不同版本字段名有差异，思路一致）：

```yaml
spawn_subagent:
  session: fresh
  tools: readonly_default
  workspace: scratch/{task_id}
  mcp: main_only
  max_turns: 12
  return: final_summary_only
```

## 踩坑点

- **隐式继承**：某些插件入口拿的是主会话上下文的引用，spawn 路径要逐个确认，别信默认值；
- **摘要偷懒**：提示词里写"顺便讲讲过程"，等于把污染原样搬回来，措辞要狠一点；
- **共享 scratch**：并行子 agent 写同一目录会互相覆盖，必须按 task id 隔离；
- **memory 工具默认开放**：除非显式需要，写长期记忆的工具应从子 agent 工具面里直接拿掉；
- **子 session 文件别急着删**：留着做审计和复现，定期归档即可。

## 可复用建议

- 把"允许子 agent 使用的工具清单"做成模板，而不是每次 spawn 手写一遍；
- 排查污染有个快速办法：grep 主 session 文件里有没有非本会话 id 的追加行，有就说明某条 spawn 路径漏了隔离；
- 子会话文件命名带上 task id 和时间戳，事后定位快很多。

## 总结

Session 隔离的本质是把"过程"和"结论"分开：过程留在子会话里自生自灭，结论以最小体积进入主会话。OpenClaw 提供的钩子是够用的，剩下的是纪律问题——每次 spawn 前问一句：这个子 agent 的中间产物，主会话真的需要吗？

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/c836c7afe2c81d9a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/785e675ba33877e9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/09cc426983397a0a.png)

