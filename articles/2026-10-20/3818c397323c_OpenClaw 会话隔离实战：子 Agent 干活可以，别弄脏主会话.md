---
title: OpenClaw 会话隔离实战：子 Agent 干活可以，别弄脏主会话
feedId: 42069
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

用 OpenClaw 跑长任务时，主会话承载着完整历史：系统提示、工具调用结果、你的每次纠偏。子 Agent（subagent / task）适合干脏活——全库检索、批量重构、跑测试。默认情况下子 Agent 有独立上下文，但**"上下文独立"不等于"不污染"**。

## 问题：污染从哪来

实践中常见三类：

1. **上下文回灌**：子 Agent 把几十次工具调用的中间输出当成结果贴回主会话，上下文瞬间膨胀，后面每一轮都在为垃圾买单。
2. **文件系统污染**：子 Agent 继承工作目录，把 scratch 文件、临时分支、未提交改动留在仓库里，主 Agent 接手时面对一堆"未知状态"。
3. **共享状态污染**：子 Agent 触发了写记忆、改配置、发通知之类的插件 hook，副作用落在了主会话头上。

## 做法：拆成三层隔离

**第一层：上下文隔离。** 派发子 Agent 时显式声明隔离模式，只给任务提示词，不带主会话历史：

```json
{
  "agent": "explore",
  "isolation": "session",
  "max_turns": 12,
  "return": "summary"
}
```

字段名以你本地版本为准，核心是三点：fresh session、turn/token 预算、只回传 summary。

**第二层：工作区隔离。** 涉及写文件的子 Agent，用独立 worktree 或临时目录跑：

```json
{ "workspace": { "mode": "worktree", "cleanup": "on-success" } }
```

主会话拿到的不是"被改过的仓库"，而是一份 diff 和一个可合并的分支，合并与否由主会话决定。

**第三层：回传契约。** 在任务提示词里写死返回格式："只返回变更文件列表、关键决策、遗留风险，各不超过 10 行"。没有契约的子 Agent 会把思考过程全倒回来。

**验证方法很朴素**：跑完一次，直接翻主会话的 transcript 文件，确认里面没有子 Agent 的工具调用记录；再对比 context 用量。隔离生效的标志是——主会话里除了你的指令和最终 summary，什么都没有。

## 踩坑点

- **把隔离当权限边界**。session 隔离是卫生措施，子 Agent 默认带着同一套凭证和 MCP 连接。要收紧权限请用 sandbox / allowlist，别指望隔离兜底。
- **MCP server 图省事全局共享**。某个子 Agent 把工具输出刷屏，输出虽然留在子会话，但连接池和限流是共享的，会把主会话的工具调用一起拖垮。重 MCP 的任务单独起实例。
- **并行子 Agent 共写一个 worktree**。两个任务同时改同一文件，merge 时互相覆盖。并行度上去之前，先保证一人一个工作区。
- **忘了 cleanup 条件**。`on-failure` 留失败现场方便排查没问题，但要配定期清扫，否则临时目录比任务攒得还快。
- **summary 本身超长**。预算要同时管住过程（max_turns）和结果（return 长度），只管一头等于没管。

## 可复用建议

- 派发前先问三个问题：它需要看主会话历史吗？它会写文件吗？它成功/失败分别留下什么？答不清楚就先别派。
- 把子 Agent 的任务提示词模板化，回传格式写进模板。团队内统一格式，主会话的下游解析才稳定。
- 定期 review 一次 sessions 目录下的会话文件，污染往往发生在你看不见的地方。
- 排查污染的顺序：先 diff transcript，再看 worktree，最后查 hook 日志，别反着来。

## 总结

session 隔离不是一个开关，而是"上下文、工作区、回传"三层各自的约束。主会话应该像主干道：只有任务指令和成品经过，施工全部发生在围挡里。做到这一点，长任务的可控性会有肉眼可见的改善。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/e1c8a9019981396f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/e928c63763cfbdae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/b775555ba74d5468.png)

