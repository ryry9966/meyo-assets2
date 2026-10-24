---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 42452
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

在 OpenClaw 里跑多步自动化时，主 Agent 常常要派活：让子 Agent 去抓数据、跑批处理、调用 MCP 工具。我们最初偷懒，子 Agent 直接复用主会话上下文跑。跑了一两天，问题就暴露了。

## 问题

- 主会话历史里混进子 Agent 的几十条工具调用轨迹，上下文快速膨胀，compaction 提前触发，最开始的原始指令反而被压缩掉；
- 主 Agent 在后续轮次会“记得”子 Agent 的中间结果，做出不该有的关联判断；
- 两个子任务写同一个 workspace，文件互相覆盖。

一句话概括：主会话应该只保留决策，不该保留子任务的执行过程。

## 做法

1. **子 Agent 一律独立 session。** spawn 时显式指定新 session ID（建议带任务名 + 时间戳），不要复用上一次的名字。
2. **约定返回契约。** 子 Agent 最终回复只给结构化结果（一段 JSON 或几行摘要），过程细节留在它自己的 session 里。
3. **workspace 分离。** 子 Agent 写临时目录，主 Agent 只读最终产物；需要共享的文件显式 copy 回主目录。
4. **生命周期明确。** 只有需要跨轮续跑的长任务才保留子 session，否则跑完即弃。
5. **留排查线索。** 把子 session ID 以一行记录写进主会话，出问题时可以回放。

最小示意（字段名以你所用版本文档为准）：

```
spawn({
  task: "抓取三张日报表",
  session: { fresh: true, id: "report-0601" },
  workspace: "tmp/report/",
  reply: { format: "json", maxTokens: 300 }
})
```

## 踩坑点

- **复用固定名字的子 session**：看起来像“记忆”，实际是旧上下文在渗漏。典型现象是子 Agent 第二次跑会引用第一次的结论。
- **把主会话整段塞进子 Agent 的 prompt**：隔离等于没做，token 照样爆炸。只传任务描述 + 必要文件路径。
- **子 Agent 直接写共享目录**：主 Agent 拿到半成品文件，下游任务全乱。必须走“临时目录 + 显式回传”。
- **隔离后子 Agent 崩了，主会话只看到空结果**：返回契约里强制包含 `status` 和 `error` 摘要，别让失败静默。
- **MCP 工具 server 是共享的**：多个子 Agent 挂同一个 server 时，连接和鉴权作用域共用，别在子任务里做会改全局状态的操作。
- **compaction 只保护主会话**：子 session 长期保留一样会膨胀，设 TTL 或定期清理。

## 可复用建议

- **验证判据**：同一任务跑两次，diff 两次的主会话记录，几乎一致才说明隔离是干净的。
- **定期看 session 文件体积**：异常大的那个，八成是漏了返回契约或没设生命周期。
- **把“子 Agent 只回 JSON”写进 spawn 模板**，靠模板约束，别靠口头约定。

## 总结

session 隔离省的不只是 token，更重要的是让主 Agent 退回协调者角色：只看结论、做决策，执行细节留在子 session 里，可回放、可清理。这套做法在我们日报自动化里落地后，主会话上下文体积降了一个量级，排查问题的成本也明显下降。如果你还在让子 Agent 和主会话共用上下文，建议先从“独立 session + 返回契约”这两步改起，收益最快。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/bed36939a6e33ce3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/a970108b450bd298.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/434e37a38cc72c99.png)

